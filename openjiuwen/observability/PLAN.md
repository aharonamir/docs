# Observability Metrics & Cost Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Last updated:** 2026-09-02
**Status:** revised roadmap — metrics and cost only (standalone tracing + trajectory already landed)

**Goal:** Add disabled-by-default OTel metrics and best-effort cost estimation/aggregation on top of the existing observability runtime.

**Architecture:** A new `MetricsRecorder` (MeterProvider + PeriodicExportingMetricReader) is owned by the existing `ObservabilityRuntime` and switched off unless `metrics_enabled=True`. Emission points live in the existing `OtelCallbackHandler` (LLM usage/duration, tool duration/errors) and `AgentObservabilityRail` (iteration duration/errors). Cost is split into (a) provider-cost passthrough (already present) and (b) a versioned pricing table that fills cost when the provider omits it, plus a trace-keyed rollup stamped on root spans.

**Tech Stack:** `opentelemetry-sdk` (MeterProvider, metrics), `opentelemetry-exporter-otlp-proto-grpc/-http` (already pinned), `pydantic` (config).

---

## As-Built Context (why this plan is small)

This plan supersedes RFC #426 and PR #427. The observability layer was restructured in August 2026:

- Shared runtime: `openjiuwen/extensions/observability/` (`runtime.py` owns the `TracerProvider`; `config.py` holds `ObservabilityConfig`; `setup.py` is the facade; `demand.py` coordinates the single process-global provider across `agent`/`team` runtimes; `callback_handler.py` owns LLM/tool spans).
- Agent tier: `openjiuwen/harness/observability/` (`AgentObservabilityRail` opens `agent.{name}.task_iteration.N` / `agent.{name}.invoke`; `run_span.py` opens the `agent.{mode}.{session}` root span; `setup.py` exposes `acquire_observability`/`release_observability`).
- Team tier: `openjiuwen/agent_teams/observability/rail.py` (`TeamObservabilityRail` decorates the agent span).

Already done (out of scope here): standalone DeepAgent tracing, subagent tracing, ordered trajectory events (`OJ_TRAJECTORY_RECORD_KIND`), provider cost passthrough on spans (`OJ_GEN_AI_USAGE_{INPUT,OUTPUT,TOTAL}_COST`). Metrics and cost estimation/rollup are the remaining gaps.

## Global Constraints

- Metrics stay **disabled by default** (`metrics_enabled: bool = False`); tracing-only users must get zero extra provider/exporter side effects.
- No new pip dependencies: `opentelemetry-sdk`, `opentelemetry-api`, `opentelemetry-exporter-otlp-proto-grpc`, `opentelemetry-exporter-otlp-proto-http` are already in the `observability` extra (`pyproject.toml:99-102`).
- Optional metrics must never raise into the model/tool/agent execution path — every emission is wrapped in `try/except` + `logger.warning`, mirroring `SafeSpanProcessor`.
- Cost estimates are **versioned** and **unknown models are marked, not guessed**.
- PEP 585 / PEP 604 types (`list[X]`, `X | None`), no `typing.Optional/List/Dict`.
- Tests: `make test TESTFLAGS="tests/unit_tests/extensions/observability/"` (see Verification Commands).

## File Structure

| File | Responsibility |
| --- | --- |
| `openjiuwen/extensions/observability/config.py` | Add `metrics_enabled`, `metrics_endpoint`, `metrics_exporter` |
| `openjiuwen/extensions/observability/metrics.py` | `MetricsRecorder` + label constants (new) |
| `openjiuwen/extensions/observability/runtime.py` | Own + init/shutdown the recorder |
| `openjiuwen/extensions/observability/setup.py` | Expose `is_metrics_enabled`/`get_metrics_recorder` |
| `openjiuwen/extensions/observability/callback_handler.py` | Emit LLM/tool metrics |
| `openjiuwen/harness/observability/rail.py` | Emit iteration duration/error metrics |
| `openjiuwen/extensions/observability/cost_tracker.py` | Versioned pricing table + estimator (new) |
| `openjiuwen/extensions/observability/usage_aggregation.py` | Trace-keyed rollup accumulator (new) |
| `deploy/observability/otel-collector-config.yaml` | Metrics pipeline + Prometheus exporter |
| `deploy/observability/docker-compose.yml` | Prometheus service |
| Tests: `tests/unit_tests/extensions/observability/test_metrics.py`, `test_cost_tracker.py`, `test_usage_aggregation.py`, `test_metrics_emission.py` (new); edits to `test_runtime.py` | Coverage |

---

## Part A — OTel Metrics

### Task 1: Metrics config fields

**Files:**
- Modify: `openjiuwen/extensions/observability/config.py`

**Interfaces:**
- Produces: `ObservabilityConfig.metrics_enabled: bool`, `.metrics_endpoint: str`, `.metrics_exporter: Literal["otlp_grpc", "otlp_http", "console"]`

- [ ] **Step 1: Write the failing test**

Create `tests/unit_tests/extensions/observability/test_metrics.py`:

```python
# coding: utf-8
from openjiuwen.extensions.observability.config import ObservabilityConfig


def test_metrics_disabled_by_default():
    cfg = ObservabilityConfig()
    assert cfg.metrics_enabled is False


def test_metrics_fields_defaults():
    cfg = ObservabilityConfig(metrics_enabled=True)
    assert cfg.metrics_endpoint == ""
    assert cfg.metrics_exporter == "otlp_grpc"


def test_metrics_exporter_rejects_unknown_value():
    import pytest
    from pydantic import ValidationError
    with pytest.raises(ValidationError):
        ObservabilityConfig(metrics_exporter="bogus")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_metrics.py"`
Expected: FAIL — `AttributeError`/`ValidationError` (fields absent).

- [ ] **Step 3: Add the fields**

In `ObservabilityConfig` (after `file_retention_days`):

```python
    # metrics
    metrics_enabled: bool = False
    metrics_endpoint: str = ""
    metrics_exporter: Literal["otlp_grpc", "otlp_http", "console"] = "otlp_grpc"
```

Update the class docstring to document the three fields: `metrics_enabled` master switch (False → no MeterProvider), `metrics_endpoint` OTLP endpoint (empty → fall back to `endpoint`), `metrics_exporter` backend.

- [ ] **Step 4: Run test to verify it passes**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_metrics.py"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add openjiuwen/extensions/observability/config.py tests/unit_tests/extensions/observability/test_metrics.py
git commit -m "feat(observability): add metrics config fields"
```

### Task 2: MetricsRecorder

**Files:**
- Create: `openjiuwen/extensions/observability/metrics.py`
- Test: `tests/unit_tests/extensions/observability/test_metrics.py` (extend)

**Interfaces:**
- Produces:
  - `MetricsRecorder(config) -> MetricsRecorder`
  - `.record_llm_usage(agent_id: str, model: str, prompt_tokens: int, completion_tokens: int) -> None`
  - `.record_llm_duration(agent_id: str, model: str, duration_ms: float) -> None`
  - `.record_tool_duration(tool_name: str, agent_id: str, duration_ms: float) -> None`
  - `.record_tool_error(tool_name: str, agent_id: str) -> None`
  - `.record_iteration_duration(agent_id: str, team_id: str, duration_ms: float) -> None`
  - `.record_iteration_error(agent_id: str, team_id: str) -> None`
  - `.shutdown() -> None`
  - module `get_metrics_recorder() -> MetricsRecorder | None`, `set_metrics_recorder(rec) -> None`, `is_metrics_enabled() -> bool`

- [ ] **Step 1: Write the failing test**

Append to `test_metrics.py`:

```python
from openjiuwen.extensions.observability import metrics as metrics_mod


def test_recorder_builds_instruments():
    cfg = ObservabilityConfig(metrics_enabled=True, metrics_exporter="console")
    rec = metrics_mod.MetricsRecorder(cfg)
    try:
        assert rec._llm_token_usage.name == "llm.token_usage"
        assert rec._llm_call_duration.name == "llm.call.duration"
        assert rec._tool_call_duration.name == "tool.call.duration"
        assert rec._tool_call_errors.name == "tool.call.errors"
        assert rec._iteration_duration.name == "deepagent.task.iteration.duration"
        assert rec._iteration_errors.name == "deepagent.task.iteration.errors"
    finally:
        rec.shutdown()


def test_recorder_record_methods_do_not_raise(monkeypatch):
    cfg = ObservabilityConfig(metrics_enabled=True, metrics_exporter="console")
    rec = metrics_mod.MetricsRecorder(cfg)
    try:
        rec.record_llm_usage("agent-1", "gpt-4o", 10, 20)
        rec.record_llm_duration("agent-1", "gpt-4o", 123.0)
        rec.record_tool_duration("bash", "agent-1", 50.0)
        rec.record_tool_error("bash", "agent-1")
        rec.record_iteration_duration("agent-1", "team-1", 400.0)
        rec.record_iteration_error("agent-1", "team-1")
    finally:
        rec.shutdown()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_metrics.py"`
Expected: FAIL — module `metrics` not found / `MetricsRecorder` not defined.

- [ ] **Step 3: Write the module**

```python
# coding: utf-8
# Copyright (c) Huawei Technologies Co., Ltd. 2026. All rights reserved.

"""Disabled-by-default OTel metrics for the observability subsystem."""

from __future__ import annotations

import threading

from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import (
    ConsoleMetricExporter,
    PeriodicExportingMetricReader,
)

from openjiuwen.core.common.logging import logger
from openjiuwen.extensions.observability.config import ObservabilityConfig

_LABEL_MODEL = "model"
_LABEL_AGENT_ID = "agent_id"
_LABEL_KIND = "kind"
_LABEL_TOOL_NAME = "tool_name"
_LABEL_TEAM_ID = "team_id"

_METER_NAME = "openjiuwen.extensions.observability"

_recorder: MetricsRecorder | None = None
_recorder_lock = threading.RLock()


def set_metrics_recorder(rec: MetricsRecorder | None) -> None:
    global _recorder
    with _recorder_lock:
        _recorder = rec


def get_metrics_recorder() -> MetricsRecorder | None:
    with _recorder_lock:
        return _recorder


def is_metrics_enabled() -> bool:
    return get_metrics_recorder() is not None


def _build_meter_provider(config: ObservabilityConfig) -> MeterProvider:
    if config.metrics_exporter == "console":
        exporter = ConsoleMetricExporter()
    elif config.metrics_exporter == "otlp_http":
        from opentelemetry.exporter.otlp.proto.http.metric_exporter import OTLPMetricExporter
        exporter = OTLPMetricExporter(endpoint=config.metrics_endpoint or config.endpoint)
    else:
        from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
        exporter = OTLPMetricExporter(
            endpoint=config.metrics_endpoint or config.endpoint,
            insecure=True,
        )
    reader = PeriodicExportingMetricReader(exporter)
    return MeterProvider(metric_readers=[reader])


class MetricsRecorder:
    """Own one ``MeterProvider`` and the high-level record API."""

    def __init__(self, config: ObservabilityConfig) -> None:
        self._provider = _build_meter_provider(config)
        # Do NOT call ``metrics.set_meter_provider``: this recorder is
        # self-contained (it uses ``self._provider.get_meter`` directly), and
        # touching the API-level global would override any coexisting meter
        # provider and break tests that build two recorders.
        meter = self._provider.get_meter(_METER_NAME)
        self._llm_token_usage = meter.create_counter(
            "llm.token_usage",
            unit="1",
            description="Prompt and completion tokens per LLM call",
        )
        self._llm_call_duration = meter.create_histogram(
            "llm.call.duration",
            unit="ms",
            description="End-to-end LLM call latency",
        )
        self._tool_call_duration = meter.create_histogram(
            "tool.call.duration",
            unit="ms",
            description="Tool execution latency",
        )
        self._tool_call_errors = meter.create_counter(
            "tool.call.errors",
            unit="1",
            description="Tool calls that raised or reported failure",
        )
        self._iteration_duration = meter.create_histogram(
            "deepagent.task.iteration.duration",
            unit="ms",
            description="Task-loop iteration latency",
        )
        self._iteration_errors = meter.create_counter(
            "deepagent.task.iteration.errors",
            unit="1",
            description="Task-loop iterations that ended in error",
        )

    def _guarded(self, fn) -> None:
        try:
            fn()
        except Exception as exc:
            logger.warning("otel: metrics record failed - {}", exc)

    def record_llm_usage(self, agent_id: str, model: str, prompt_tokens: int, completion_tokens: int) -> None:
        self._guarded(lambda: self._llm_token_usage.add(
            int(prompt_tokens),
            {_LABEL_MODEL: model, _LABEL_AGENT_ID: agent_id, _LABEL_KIND: "prompt"},
        ))
        self._guarded(lambda: self._llm_token_usage.add(
            int(completion_tokens),
            {_LABEL_MODEL: model, _LABEL_AGENT_ID: agent_id, _LABEL_KIND: "completion"},
        ))

    def record_llm_duration(self, agent_id: str, model: str, duration_ms: float) -> None:
        self._guarded(lambda: self._llm_call_duration.record(
            float(duration_ms), {_LABEL_MODEL: model, _LABEL_AGENT_ID: agent_id},
        ))

    def record_tool_duration(self, tool_name: str, agent_id: str, duration_ms: float) -> None:
        self._guarded(lambda: self._tool_call_duration.record(
            float(duration_ms), {_LABEL_TOOL_NAME: tool_name, _LABEL_AGENT_ID: agent_id},
        ))

    def record_tool_error(self, tool_name: str, agent_id: str) -> None:
        self._guarded(lambda: self._tool_call_errors.add(
            1, {_LABEL_TOOL_NAME: tool_name, _LABEL_AGENT_ID: agent_id},
        ))

    def record_iteration_duration(self, agent_id: str, team_id: str, duration_ms: float) -> None:
        self._guarded(lambda: self._iteration_duration.record(
            float(duration_ms), {_LABEL_AGENT_ID: agent_id, _LABEL_TEAM_ID: team_id},
        ))

    def record_iteration_error(self, agent_id: str, team_id: str) -> None:
        self._guarded(lambda: self._iteration_errors.add(
            1, {_LABEL_AGENT_ID: agent_id, _LABEL_TEAM_ID: team_id},
        ))

    def shutdown(self) -> None:
        with _recorder_lock:
            try:
                self._provider.shutdown()
            except Exception as exc:
                logger.warning("otel: metrics shutdown failed - {}", exc)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_metrics.py"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add openjiuwen/extensions/observability/metrics.py tests/unit_tests/extensions/observability/test_metrics.py
git commit -m "feat(observability): add MetricsRecorder with core instruments"
```

### Task 3: Runtime integration

**Files:**
- Modify: `openjiuwen/extensions/observability/runtime.py`
- Modify: `openjiuwen/extensions/observability/setup.py`
- Test: `tests/unit_tests/extensions/observability/test_runtime.py` (extend)

**Interfaces:**
- Consumes: `MetricsRecorder` from Task 2.
- Produces: `ObservabilityRuntime.initialize()` creates + sets the recorder when `config.metrics_enabled`; `.shutdown()` clears it; `setup.get_metrics_recorder()` and `setup.is_metrics_enabled()` passthroughs.

- [ ] **Step 1: Write the failing test**

Append to `test_runtime.py`:

```python
def test_metrics_initialized_when_enabled(reset_observability_runtime):
    from openjiuwen.extensions.observability.setup import (
        get_metrics_recorder,
        init_observability,
        is_metrics_enabled,
        shutdown_observability,
    )
    cfg = ObservabilityConfig(metrics_enabled=True, metrics_exporter="console")
    init_observability(cfg)
    try:
        assert is_metrics_enabled()
        assert get_metrics_recorder() is not None
    finally:
        shutdown_observability()
    assert not is_metrics_enabled()


def test_metrics_not_initialized_by_default(reset_observability_runtime):
    from openjiuwen.extensions.observability.setup import (
        init_observability,
        is_metrics_enabled,
        shutdown_observability,
    )
    init_observability(ObservabilityConfig())
    try:
        assert not is_metrics_enabled()
    finally:
        shutdown_observability()
```

`test_runtime.py` has **no** `reset_observability_runtime` fixture — add it, plus a regression test for the already-initialized case (see M3):

```python
import pytest


@pytest.fixture
def reset_observability_runtime():
    from openjiuwen.extensions.observability import metrics as _metrics
    from openjiuwen.extensions.observability.demand import reset_observability_demands
    from openjiuwen.extensions.observability.setup import shutdown_observability
    shutdown_observability()
    _metrics.set_metrics_recorder(None)
    reset_observability_demands()
    yield
    shutdown_observability()
    _metrics.set_metrics_recorder(None)


def test_metrics_initialized_on_second_runtime_when_enabled(reset_observability_runtime):
    from openjiuwen.extensions.observability.setup import (
        get_metrics_recorder,
        init_observability,
        is_metrics_enabled,
        shutdown_observability,
    )
    init_observability(ObservabilityConfig())  # first init: tracing only
    assert not is_metrics_enabled()
    init_observability(ObservabilityConfig(metrics_enabled=True, metrics_exporter="console"))
    try:
        assert is_metrics_enabled()
        assert get_metrics_recorder() is not None
    finally:
        shutdown_observability()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_runtime.py"`
Expected: FAIL — `get_metrics_recorder`/`is_metrics_enabled` not exported.

- [ ] **Step 3: Implement**

In `runtime.py` `ObservabilityRuntime.__init__`, add `self._metrics_recorder: Any | None = None`.

The recorder is independent of the trace provider, so it must be created in **both** branches of `initialize()`:

1. **First-init path** (after the provider is set up, just before `self._callback_handler = ...`):

```python
if config.metrics_enabled:
    from openjiuwen.extensions.observability.metrics import MetricsRecorder, set_metrics_recorder
    self._metrics_recorder = MetricsRecorder(config)
    set_metrics_recorder(self._metrics_recorder)
else:
    set_metrics_recorder(None)
```

2. **Already-initialized path** (the early-return branch where `self._provider is not None`, `runtime.py:124-132`). Without this, a team runtime that initializes first with metrics off would leave a later `metrics_enabled=True` acquisition silently without metrics:

```python
if self._provider is not None:
    if span_exporter_override is not None:
        self._provider.add_span_processor(SimpleSpanProcessor(span_exporter_override))
    self.add_span_processors(additional_span_processors)
    # metrics: independent of the trace provider; build lazily if requested
    if config.metrics_enabled and self._metrics_recorder is None:
        from openjiuwen.extensions.observability.metrics import MetricsRecorder, set_metrics_recorder
        self._metrics_recorder = MetricsRecorder(config)
        set_metrics_recorder(self._metrics_recorder)
    return
```

In the rollback path (the `except Exception:` block), and in `shutdown()`, add:

```python
from openjiuwen.extensions.observability.metrics import set_metrics_recorder
if self._metrics_recorder is not None:
    self._metrics_recorder.shutdown()
self._metrics_recorder = None
set_metrics_recorder(None)
```

In `setup.py`, add:

```python
def get_metrics_recorder() -> Any:
    from openjiuwen.extensions.observability.metrics import get_metrics_recorder as _get
    return _get()

def is_metrics_enabled() -> bool:
    from openjiuwen.extensions.observability.metrics import is_metrics_enabled as _is
    return _is()
```

and add both to `__all__`.

- [ ] **Step 4: Run test to verify it passes**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_runtime.py"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add openjiuwen/extensions/observability/runtime.py openjiuwen/extensions/observability/setup.py tests/unit_tests/extensions/observability/test_runtime.py
git commit -m "feat(observability): own metrics recorder in ObservabilityRuntime"
```

### Task 4: Callback + rail emission

**Files:**
- Modify: `openjiuwen/extensions/observability/callback_handler.py`
- Modify: `openjiuwen/harness/observability/rail.py`
- Test: `tests/unit_tests/extensions/observability/test_metrics_emission.py` (new)

**Interfaces:**
- Consumes: `metrics.get_metrics_recorder()`.
- Produces: `OtelCallbackHandler._emit_llm_metrics(state)` called at LLM close; `_emit_tool_metrics(tool_name, agent_id, duration_ms, is_error)` at tool close; `AgentObservabilityRail` emits `deepagent.task.iteration.*` at `after_task_iteration`/error.

- [ ] **Step 1: Write the failing test**

Create `test_metrics_emission.py` (the handler-driving pattern mirrors `test_callback_handler.py`, which builds a `TracerProvider` + `OtelCallbackHandler(config, tracer=...)` and monkeypatches `_get_parent_context_for_llm_tool`):

```python
# coding: utf-8
from types import SimpleNamespace

from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.trace import set_span_in_context

from openjiuwen.extensions.observability import metrics as metrics_mod
from openjiuwen.extensions.observability.callback_handler import OtelCallbackHandler
from openjiuwen.extensions.observability.config import ObservabilityConfig


class _FakeMetricsRecorder:
    def __init__(self) -> None:
        self.llm_usage_calls: list[tuple] = []
        self.llm_duration_calls: list[tuple] = []
        self.tool_duration_calls: list[tuple] = []
        self.tool_error_calls: list[tuple] = []

    def record_llm_usage(self, agent_id, model, prompt, completion) -> None:
        self.llm_usage_calls.append((agent_id, model, prompt, completion))

    def record_llm_duration(self, agent_id, model, duration_ms) -> None:
        self.llm_duration_calls.append((agent_id, model, duration_ms))

    def record_tool_duration(self, tool_name, agent_id, duration_ms) -> None:
        self.tool_duration_calls.append((tool_name, agent_id, duration_ms))

    def record_tool_error(self, tool_name, agent_id) -> None:
        self.tool_error_calls.append((tool_name, agent_id))


def _handler(monkeypatch, recorder):
    provider = TracerProvider()
    tracer = provider.get_tracer("metrics-test")
    root = tracer.start_span("agent.root")
    handler = OtelCallbackHandler(
        ObservabilityConfig(enabled=True, service_name="metrics-test"),
        tracer=tracer,
    )
    monkeypatch.setattr(handler, "_get_parent_context_for_llm_tool", lambda: set_span_in_context(root))
    monkeypatch.setattr(metrics_mod, "get_metrics_recorder", lambda: recorder)
    return provider, root, handler


def test_llm_close_emits_metrics_when_enabled(monkeypatch):
    rec = _FakeMetricsRecorder()
    provider, _root, handler = _handler(monkeypatch, rec)

    span = handler._open_llm_span({"messages": [], "model": "gpt-4o"})
    assert span is not None
    span.set_attribute("gen_ai.usage.input_tokens", 10)
    span.set_attribute("gen_ai.usage.output_tokens", 20)
    span.set_attribute("gen_ai.response.model", "gpt-4o")
    span.set_attribute("gen_ai.agent.name", "agent-1")
    state = getattr(span, "otel_llm_state")

    handler._close_llm_span(state, SimpleNamespace())

    assert rec.llm_usage_calls == [("agent-1", "gpt-4o", 10, 20)]
    assert len(rec.llm_duration_calls) == 1
    provider.shutdown()


async def test_tool_error_emits_error_metric(monkeypatch):
    rec = _FakeMetricsRecorder()
    provider, _root, handler = _handler(monkeypatch, rec)
    monkeypatch.setattr(handler, "_metrics_agent_id", lambda span: "agent-1")

    await handler.on_tool_call_started(tool_name="bash", tool_id=None, inputs=None)
    await handler.on_tool_call_error(tool_name="bash", error=RuntimeError("boom"), tool_id=None)

    assert ("bash", "agent-1") in rec.tool_error_calls
    assert rec.tool_duration_calls
    provider.shutdown()
```

- [ ] **Step 2: Run test to verify it fails**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_metrics_emission.py"`
Expected: FAIL — no metrics calls recorded.

- [ ] **Step 3: Implement**

In `callback_handler.py`, add two helpers and call them at the close sites.

Duration is `(time.time_ns() - span.start_time) / 1_000_000.0`, computed **before** `span.end()`.

Helper (imports `time` and `from openjiuwen.extensions.observability import metrics as _metrics`):

```python
def _metrics_agent_id(self, span) -> str:
    return str(
        span.attributes.get(GEN_AI_AGENT_NAME)
        or span.attributes.get(DA_AGENT_NAME)
        or span.attributes.get(GEN_AI_AGENT_ID)
        or "unknown"
    )

def _metrics_model(self, span) -> str:
    return str(
        span.attributes.get(GEN_AI_RESPONSE_MODEL)
        or span.attributes.get(GEN_AI_REQUEST_MODEL)
        or "unknown"
    )

def _emit_llm_metrics(self, state) -> None:
    rec = _metrics.get_metrics_recorder()
    if rec is None or not state.span.is_recording():
        return
    usage = state.span.attributes
    prompt = int(usage.get(GEN_AI_USAGE_INPUT_TOKENS, 0) or 0)
    completion = int(usage.get(GEN_AI_USAGE_OUTPUT_TOKENS, 0) or 0)
    if not prompt and not completion:
        prompt = int(usage.get(GEN_AI_USAGE_PROMPT_TOKENS, 0) or 0)
        completion = int(usage.get(GEN_AI_USAGE_COMPLETION_TOKENS, 0) or 0)
    agent_id = self._metrics_agent_id(state.span)
    model = self._metrics_model(state.span)
    duration_ms = (time.time_ns() - state.span.start_time) / 1_000_000.0
    rec.record_llm_usage(agent_id, model, prompt, completion)
    rec.record_llm_duration(agent_id, model, duration_ms)
```

Call `self._emit_llm_metrics(state)` inside `_close_llm_span` (before `span.end()`) — this covers both `on_llm_stream_completed` and `on_llm_invoke_output`.

For tools, add:

```python
def _emit_tool_metrics(self, tool_name: str, agent_id: str, duration_ms: float, is_error: bool) -> None:
    rec = _metrics.get_metrics_recorder()
    if rec is None:
        return
    rec.record_tool_duration(tool_name, agent_id, duration_ms)
    if is_error:
        rec.record_tool_error(tool_name, agent_id)
```

Call it in `on_tool_call_finished` (with `is_error=bool(failure_reason)` where `failure_reason = tool_failure_reason(result)`) and in `on_tool_call_error` (`is_error=True`). In both, derive `agent_id = self._metrics_agent_id(span)` and compute `duration_ms` from `span.start_time` before `span.end()` / `span.set_status(...)`.

In `harness/observability/rail.py` `AgentObservabilityRail`, add an `_emit_iteration_metrics(scope)` helper called from `after_task_iteration` before `scope.close(...)`:

```python
def _emit_iteration_metrics(self, scope, exception: BaseException | None) -> None:
    from openjiuwen.extensions.observability import metrics as _metrics
    rec = _metrics.get_metrics_recorder()
    if rec is None:
        return
    span = scope.span
    agent_id = str(span.attributes.get(DA_AGENT_NAME) or "unknown")
    team_id = str(span.attributes.get(OJ_TEAM_ID) or "")
    duration_ms = (time.time_ns() - span.start_time) / 1_000_000.0
    rec.record_iteration_duration(agent_id, team_id, duration_ms)
    if exception is not None:
        rec.record_iteration_error(agent_id, team_id)
```

Add `import time` and the `OJ_TEAM_ID` import to `rail.py` (verify `OJ_TEAM_ID` is exported from `semconv`; if not, use `"openjiuwen.team.id"` literal).

- [ ] **Step 4: Run test to verify it passes**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_metrics_emission.py tests/unit_tests/harness/observability/test_rail.py"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add openjiuwen/extensions/observability/callback_handler.py openjiuwen/harness/observability/rail.py tests/unit_tests/extensions/observability/test_metrics_emission.py
git commit -m "feat(observability): emit LLM/tool/iteration metrics"
```

### Task 5: Deploy wiring (Prometheus + collector metrics pipeline)

**Files:**
- Modify: `deploy/observability/otel-collector-config.yaml`
- Modify: `deploy/observability/docker-compose.yml`
- Create: `deploy/observability/prometheus.yml`

- [ ] **Step 1: Extend the collector config**

Add a `prometheus` exporter and a `metrics` pipeline (the `otlp` receiver already accepts metrics on 4317/4318):

```yaml
exporters:
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, transform/limit_strings]
      exporters: [otlphttp/langfuse, debug]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [prometheus, debug]
```

- [ ] **Step 2: Add Prometheus**

In `docker-compose.yml`, add a `prometheus` service and expose the collector's `8889` port:

```yaml
  otel-collector:
    ports:
      - "4317:4317"
      - "4318:4318"
      - "8889:8889"

  prometheus:
    image: prom/prometheus:v2.53.0
    container_name: langfuse-prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
```

Create `deploy/observability/prometheus.yml`:

```yaml
scrape_configs:
  - job_name: otel-collector
    scrape_interval: 10s
    static_configs:
      - targets: ["otel-collector:8889"]
```

- [ ] **Step 3: Verify the stack config parses**

Run: `docker compose -f deploy/observability/docker-compose.yml config -q` (if Docker available).
Expected: no parse errors.

- [ ] **Step 4: Commit**

```bash
git add deploy/observability/otel-collector-config.yaml deploy/observability/docker-compose.yml deploy/observability/prometheus.yml
git commit -m "feat(observability): add metrics pipeline and Prometheus"
```

---

## Part B — Cost Estimation & Aggregation

### Task 6: Cost estimator (versioned pricing table)

**Files:**
- Create: `openjiuwen/extensions/observability/cost_tracker.py`
- Test: `tests/unit_tests/extensions/observability/test_cost_tracker.py` (new)

**Interfaces:**
- Produces:
  - `ModelPrice(input_usd_per_1m, output_usd_per_1m)` (frozen dataclass)
  - `CostEstimate(input_cost, output_cost, total_cost, pricing_version, known)` (frozen dataclass)
  - `estimate_cost(model: str, prompt_tokens: int, completion_tokens: int) -> CostEstimate`
  - `register_model_prices(version: str, prices: dict[str, ModelPrice]) -> None`
  - `PRICING_VERSION: str`

- [ ] **Step 1: Write the failing test**

Create `test_cost_tracker.py`:

```python
# coding: utf-8
import pytest

from openjiuwen.extensions.observability import cost_tracker as ct
from openjiuwen.extensions.observability.cost_tracker import (
    PRICING_VERSION,
    ModelPrice,
    estimate_cost,
    register_model_prices,
)


@pytest.fixture
def reset_pricing():
    saved_prices = ct._PRICING
    saved_version = ct._VERSION
    yield
    ct._PRICING = saved_prices
    ct._VERSION = saved_version


def test_known_model_estimates_cost(reset_pricing):
    register_model_prices("test", {"my-model": ModelPrice(1.0, 2.0)})
    est = estimate_cost("my-model", 1000, 500)
    assert est.known is True
    assert est.pricing_version == "test"
    assert abs(est.input_cost - 0.001) < 1e-9
    assert abs(est.output_cost - 0.001) < 1e-9
    assert abs(est.total_cost - 0.002) < 1e-9


def test_unknown_model_is_marked_not_guessed(reset_pricing):
    register_model_prices("test", {})  # empty table: every model unknown
    est = estimate_cost("no-such-model", 1000, 500)
    assert est.known is False
    assert est.total_cost == 0.0


def test_default_table_versioned():
    assert PRICING_VERSION
```

- [ ] **Step 2: Run test to verify it fails**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_cost_tracker.py"`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

```python
# coding: utf-8
# Copyright (c) Huawei Technologies Co., Ltd. 2026. All rights reserved.

"""Best-effort, versioned model cost estimation.

Providers already return cost on many responses and those values flow straight
through to span attributes (``OJ_GEN_AI_USAGE_*_COST``). This module fills the
gap when a provider omits cost: a versioned per-1M-token price table that users
can override without editing code. Unknown models are reported with zero cost
and ``known=False`` — never guessed.
"""

from __future__ import annotations

from dataclasses import dataclass

PRICING_VERSION = "2026-09-01"


@dataclass(frozen=True)
class ModelPrice:
    input_usd_per_1m: float
    output_usd_per_1m: float


@dataclass(frozen=True)
class CostEstimate:
    input_cost: float
    output_cost: float
    total_cost: float
    pricing_version: str
    known: bool


_DEFAULT_PRICES: dict[str, ModelPrice] = {
    # Placeholder illustrative entries; fill from the current provider price
    # pages at implementation time and keep them conservative.
}

_PRICING: dict[str, ModelPrice] = dict(_DEFAULT_PRICES)
_VERSION = PRICING_VERSION


def register_model_prices(version: str, prices: dict[str, ModelPrice]) -> None:
    """Replace the active table (full replacement, not a merge).

    Callers who want to layer overrides must pass the complete table. This
    keeps a registered version a single authoritative snapshot.
    """
    global _PRICING, _VERSION
    _PRICING = dict(prices)
    _VERSION = version


def estimate_cost(model: str, prompt_tokens: int, completion_tokens: int) -> CostEstimate:
    price = _PRICING.get(model)
    if price is None:
        return CostEstimate(0.0, 0.0, 0.0, _VERSION, False)
    input_cost = prompt_tokens / 1_000_000 * price.input_usd_per_1m
    output_cost = completion_tokens / 1_000_000 * price.output_usd_per_1m
    return CostEstimate(input_cost, output_cost, input_cost + output_cost, _VERSION, True)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_cost_tracker.py"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add openjiuwen/extensions/observability/cost_tracker.py tests/unit_tests/extensions/observability/test_cost_tracker.py
git commit -m "feat(observability): add versioned cost estimator"
```

### Task 7: Fill cost when provider omits it

**Files:**
- Modify: `openjiuwen/extensions/observability/callback_handler.py`
- Test: `tests/unit_tests/extensions/observability/test_metrics_emission.py` (extend)

**Interfaces:**
- Consumes: `estimate_cost` from Task 6.
- Produces: `_record_usage_attrs` writes `OJ_GEN_AI_USAGE_*_COST` even when `usage` has no `*_cost` fields.

- [ ] **Step 1: Write the failing test**

In `test_metrics_emission.py`, drive a non-streaming LLM close with a usage object that has `input_tokens=1000`, `output_tokens=500`, `model_name="my-model"` but **no** cost fields, after `register_model_prices("test", {"my-model": ModelPrice(1.0, 2.0)})`. Assert the resulting span has `OJ_GEN_AI_USAGE_TOTAL_COST == 0.002`.

- [ ] **Step 2: Run test to verify it fails**

Expected: FAIL — cost attribute absent (0.0).

- [ ] **Step 3: Implement**

In `_record_usage_attrs`, after the existing cost passthrough loop (around `callback_handler.py:1388-1394`), add:

```python
provider_cost = (
    float(getattr(usage, "input_cost", 0) or 0)
    + float(getattr(usage, "output_cost", 0) or 0)
    + float(getattr(usage, "total_cost", 0) or 0)
)
model = str(getattr(usage, "model_name", "") or state.span.attributes.get(GEN_AI_RESPONSE_MODEL) or "")
if not provider_cost and model:
    from openjiuwen.extensions.observability.cost_tracker import estimate_cost
    prompt = int(state.span.attributes.get(GEN_AI_USAGE_INPUT_TOKENS, 0) or 0)
    completion = int(state.span.attributes.get(GEN_AI_USAGE_OUTPUT_TOKENS, 0) or 0)
    est = estimate_cost(model, prompt, completion)
    if est.known and not (skip_existing and OJ_GEN_AI_USAGE_TOTAL_COST in state.span.attributes):
        state.span.set_attribute(OJ_GEN_AI_USAGE_INPUT_COST, est.input_cost)
        state.span.set_attribute(OJ_GEN_AI_USAGE_OUTPUT_COST, est.output_cost)
        state.span.set_attribute(OJ_GEN_AI_USAGE_TOTAL_COST, est.total_cost)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_metrics_emission.py"`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add openjiuwen/extensions/observability/callback_handler.py tests/unit_tests/extensions/observability/test_metrics_emission.py
git commit -m "feat(observability): estimate cost when provider omits it"
```

### Task 8: Rollup aggregation on root spans

**Files:**
- Create: `openjiuwen/extensions/observability/usage_aggregation.py`
- Modify: `openjiuwen/extensions/observability/callback_handler.py`
- Modify: `openjiuwen/harness/observability/run_span.py` (stamp at root close)
- Test: `tests/unit_tests/extensions/observability/test_usage_aggregation.py` (new)

**Interfaces:**
- Consumes: `estimate_cost`/span cost attrs from Task 7.
- Produces:
  - `UsageAccumulator` (thread-safe, keyed by `trace_id`): `accumulate_llm(trace_id, prompt_tokens, completion_tokens, cost)`, `accumulate_tool(trace_id, is_error)`, `snapshot(trace_id) -> dict[str, Any]`, `clear(trace_id)`
  - module `get_accumulator() -> UsageAccumulator` singleton

- [ ] **Step 1: Write the failing test**

Create `test_usage_aggregation.py`:

```python
# coding: utf-8
from openjiuwen.extensions.observability.usage_aggregation import UsageAccumulator


def test_accumulate_and_snapshot():
    acc = UsageAccumulator()
    tid = 12345
    acc.accumulate_llm(tid, prompt=10, completion=20, cost=0.5)
    acc.accumulate_llm(tid, prompt=5, completion=3, cost=0.1)
    acc.accumulate_tool(tid, is_error=False)
    acc.accumulate_tool(tid, is_error=True)
    snap = acc.snapshot(tid)
    assert snap["prompt_tokens"] == 15
    assert snap["completion_tokens"] == 23
    assert snap["tool_calls"] == 2
    assert snap["tool_errors"] == 1
    assert abs(snap["cost"] - 0.6) < 1e-9
    acc.clear(tid)
    assert acc.snapshot(tid) == {}


def test_snapshot_unknown_trace_is_empty():
    assert UsageAccumulator().snapshot(999) == {}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_usage_aggregation.py"`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

```python
# coding: utf-8
# Copyright (c) Huawei Technologies Co., Ltd. 2026. All rights reserved.

"""Trace-keyed rollup of LLM/tool facts for root-span stamping."""

from __future__ import annotations

import threading

_ACCUMULATOR: UsageAccumulator | None = None
_LOCK = threading.RLock()


class UsageAccumulator:
    def __init__(self) -> None:
        self._data: dict[int, dict[str, float]] = {}
        self._lock = threading.RLock()

    def _entry(self, trace_id: int) -> dict[str, float]:
        return self._data.setdefault(
            trace_id,
            {"prompt_tokens": 0, "completion_tokens": 0, "cost": 0.0, "tool_calls": 0, "tool_errors": 0},
        )

    def accumulate_llm(self, trace_id: int, *, prompt: int, completion: int, cost: float) -> None:
        with self._lock:
            e = self._entry(trace_id)
            e["prompt_tokens"] += prompt
            e["completion_tokens"] += completion
            e["cost"] += cost

    def accumulate_tool(self, trace_id: int, *, is_error: bool) -> None:
        with self._lock:
            e = self._entry(trace_id)
            e["tool_calls"] += 1
            if is_error:
                e["tool_errors"] += 1

    def snapshot(self, trace_id: int) -> dict[str, float]:
        with self._lock:
            return dict(self._data.get(trace_id, {}))

    def clear(self, trace_id: int) -> None:
        with self._lock:
            self._data.pop(trace_id, None)


def get_accumulator() -> UsageAccumulator:
    global _ACCUMULATOR
    with _LOCK:
        if _ACCUMULATOR is None:
            _ACCUMULATOR = UsageAccumulator()
        return _ACCUMULATOR
```

- [ ] **Step 4: Wire accumulation + stamping**

In `callback_handler.py`, at the end of `_record_usage_attrs` (and the LLM close path), call `get_accumulator().accumulate_llm(trace_id, prompt=..., completion=..., cost=<span cost>)`, reading the final values from `state.span.attributes`. In tool close/error, call `accumulate_tool(trace_id, is_error=...)`. Use `state.span.context.trace_id` as the key.

In `run_span.py` `close_agent_run_span`, `handle` **is** the root span and `trace_id` is already computed as a local (`run_span.py:~274`, `trace_id = getattr(getattr(handle, "context", None), "trace_id", None)`). Place the stamping after that line and before the root's final `end()`:

```python
from openjiuwen.extensions.observability.usage_aggregation import get_accumulator
if trace_id is not None:
    snap = get_accumulator().snapshot(trace_id)
    if snap:
        handle.set_attribute("openjiuwen.run.total_prompt_tokens", int(snap["prompt_tokens"]))
        handle.set_attribute("openjiuwen.run.total_completion_tokens", int(snap["completion_tokens"]))
        handle.set_attribute("openjiuwen.run.total_tool_calls", int(snap["tool_calls"]))
        handle.set_attribute("openjiuwen.run.estimated_cost_usd", snap["cost"])
        get_accumulator().clear(trace_id)
```

**Naming:** single-agent roots use the `openjiuwen.run.*` namespace; team roots use the RFC's `agentteam.task.*` namespace. Both are drained from the same trace-keyed `UsageAccumulator` and cleared. The single-agent path stamps in `close_agent_run_span` (`run_span.py`); the symmetric team path drains/stamps/clears in `finalize_trace` (`agent_teams/observability/span_context.py`) — the team root's real close point, not `TeamObservabilityRail` (which only decorates agent spans). A shared `drain_rollup()` accessor makes both finalizers snapshot-and-clear atomically so neither can leak an accumulator entry.

- [ ] **Step 5: Run test to verify it passes**

Run: `make test TESTFLAGS="tests/unit_tests/extensions/observability/test_usage_aggregation.py tests/unit_tests/extensions/observability/test_metrics_emission.py tests/unit_tests/harness/observability/test_run_span.py"`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add openjiuwen/extensions/observability/usage_aggregation.py openjiuwen/extensions/observability/callback_handler.py openjiuwen/harness/observability/run_span.py tests/unit_tests/extensions/observability/test_usage_aggregation.py
git commit -m "feat(observability): aggregate usage and stamp root spans"
```

---

## Verification Commands

```bash
make test TESTFLAGS="tests/unit_tests/extensions/observability/"
make test TESTFLAGS="tests/unit_tests/harness/observability/"
make test TESTFLAGS="tests/unit_tests/agent_teams/observability/"
make check
make type-check
```

## Open Design Point (deferred, not in this plan)

Session-close lifecycle: `SessionEvents.AGENT_SESSION_CREATED` still has no symmetric close event. The run-span model (`open_agent_run_span`/`close_agent_run_span`) already bounds single-agent roots; if session-level grouping is wanted later, add a `SESSION_CLOSED` event at its real lifecycle owner. Do not bundle it into metrics/cost.

## Known Limitation (accepted, no new channel)

The usage rollup is **process-local** (`UsageAccumulator` is a module-level singleton). With the default `spawn_mode="process"`, teammates run in separate processes, so each accumulates into its own private accumulator and the leader's `finalize_trace` only sees usage that ran in the leader process. The `agentteam.task.*` rollup on the team root is therefore **best-effort and incomplete across processes** — a deliberate, documented limitation. No cross-process aggregation channel (e.g. the `TeamEvent` stream) is added in this plan; if exact team totals are required later, wire accumulator results into the existing team event/monitor stream as a separate change.
