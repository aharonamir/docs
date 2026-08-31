# TraceHound Trajectory Improvements — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the chat-embedded TraceHound trajectory panel accurate (team LLM usage), attractive (theme tokens), and engaging (highlights, timeline, per-agent charts, flagged live updates, cross-links, i18n, Langfuse-style agent graph).

**Architecture:** Foundation-first: theme tokens + backend usage aggregation land before features built on them. All new frontend logic lives in pure modules tested with the repo's `node --test` pattern; backend replay changes are tested with pytest on `AgentWebSocketServer.__new__`. No new dependencies; all charts/graphs are hand-rolled SVG using CSS-variable theme tokens.

**Tech Stack:** React 18 + TypeScript + Vite (frontend), Python asyncio server (backend), pytest, `node --test`, Playwright (manual verification).

**Spec:** `docs/superpowers/specs/2026-08-27-tracehound-informative-plan.md`

## Global Constraints

- No hardcoded product colors in components — use `--color-trace-*` tokens via `traceTokens.ts` (`AGENTS.md` rule; inline styles may reference `var(--color-*)`).
- Chrome/Chrome-Clang 107 is the browser baseline; no newer-only web APIs.
- Follow `.prettierrc.cjs`; do not reformat untouched code.
- `tsconfig.json` has `noUnusedLocals`/`noUnusedParameters` — no dead code may remain.
- Frontend verify: `cd jiuwenswarm/channels/web/frontend && npx tsc --noEmit` then `npm run build` (web server serves `dist/`, so build is required for live checks).
- Backend tests: `cd jiuwenswarm && python -m pytest tests/unit_tests/server/<file> -v` (pytest 9.0.3 installed; no conftest).
- Web server serves pre-built `frontend/dist/` — a hard refresh of the browser is enough after `npm run build`; no backend restart needed for frontend-only tasks.
- Do not stage repo junk (`.opencode/`, `AGENTS.old.md`, `agent-loop.*`, `conversation_history/`, `docs/AutoTool_Plan.md`, `jiuwenswarm/token-optimizer/`, `openwiki/`, `tests/verify_trajectory_signal.py`, `token-optimizer/`).
- New SVG must use token colors (inline SVG referencing CSS vars is compliant).
- Historical team sessions have no usage data — UI must show `—`, never a misleading `0`, when a session has zero usage records.

---

### Task 1: Trace theme tokens + chrome migration

**Files:**
- Modify: `jiuwenswarm/channels/web/frontend/src/styles/themes/default/light.css`
- Create: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/traceTokens.ts`
- Modify: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx` (TrajectoryPanel header + TurnListView section headers only)

**Interfaces:**
- Produces: `traceTokens.ts` exports `C` (record of token-name → `var(--color-trace-*)` strings) and `cat(n: number): string` for the categorical scale. Later tasks consume `C.*` everywhere instead of hex.

- [ ] **Step 1: Add tokens to light.css**

Append inside `:root[data-theme="default"][data-color-mode="light"] { … }` (after the Feedback block, ~line 90):

```css
  /* TraceHound — trajectory observer palette (role-named, observability light) */
  --color-trace-ok: #10b981;
  --color-trace-ok-subtle: rgba(16, 185, 129, 0.1);
  --color-trace-warn: #d97706;
  --color-trace-warn-subtle: rgba(217, 119, 6, 0.12);
  --color-trace-danger: #dc2626;
  --color-trace-danger-subtle: rgba(220, 38, 38, 0.1);
  --color-trace-violet: #7c3aed;
  --color-trace-violet-subtle: rgba(124, 58, 237, 0.1);
  --color-trace-info: #2563eb;
  --color-trace-info-subtle: rgba(37, 99, 235, 0.1);
  --color-trace-teal: #0d9488;
  --color-trace-cat-1: #2563eb;
  --color-trace-cat-2: #7c3aed;
  --color-trace-cat-3: #d97706;
  --color-trace-cat-4: #0d9488;
  --color-trace-cat-5: #db2777;
  --color-trace-cat-6: #4f46e5;
  --color-trace-surface: #ffffff;
  --color-trace-surface-muted: #f7f8fa;
  --color-trace-border: #e4e7ec;
  --color-trace-border-strong: #cbd5e1;
  --color-trace-text: #101828;
  --color-trace-text-muted: #667085;
  --color-trace-text-faint: #98a2b3;
```

- [ ] **Step 2: Create `traceTokens.ts`**

```ts
/**
 * TraceHound palette: role-named CSS-variable tokens.
 * Components use C.* in inline styles; concrete values live only in light.css.
 */
export const C = {
  ok: 'var(--color-trace-ok)',
  okSubtle: 'var(--color-trace-ok-subtle)',
  warn: 'var(--color-trace-warn)',
  warnSubtle: 'var(--color-trace-warn-subtle)',
  danger: 'var(--color-trace-danger)',
  dangerSubtle: 'var(--color-trace-danger-subtle)',
  violet: 'var(--color-trace-violet)',
  violetSubtle: 'var(--color-trace-violet-subtle)',
  info: 'var(--color-trace-info)',
  infoSubtle: 'var(--color-trace-info-subtle)',
  teal: 'var(--color-trace-teal)',
  surface: 'var(--color-trace-surface)',
  surfaceMuted: 'var(--color-trace-surface-muted)',
  border: 'var(--color-trace-border)',
  borderStrong: 'var(--color-trace-border-strong)',
  text: 'var(--color-trace-text)',
  textMuted: 'var(--color-trace-text-muted)',
  textFaint: 'var(--color-trace-text-faint)',
} as const;

/** Categorical color for agents/query types (1-based, cycles). */
export function cat(n: number): string {
  const i = ((n - 1) % 6) + 1;
  return `var(--color-trace-cat-${i})`;
}
```

- [ ] **Step 3: Migrate chrome colors in `index.tsx` (headers only — full migration is Task 9)**

In `TrajectoryPanel` header replace `#e5e7eb` → `C.border`, `#374151` → `C.text`; in `TurnListView` section-header rows replace `#e5e7eb`/`#f9fafb`/`#374151`/`#6b7280`/`#9ca3af` with `C.border`/`C.surfaceMuted`/`C.text`/`C.textMuted`/`C.textFaint`. Add `import { C } from './traceTokens';` at top. Leave all other hex colors for Task 9.

- [ ] **Step 4: Verify**

Run: `cd jiuwenswarm/channels/web/frontend && npx tsc --noEmit && npm run build`
Expected: tsc exit 0; build succeeds.

- [ ] **Step 5: Commit**

```bash
git add jiuwenswarm/channels/web/frontend/src/styles/themes/default/light.css \
        jiuwenswarm/channels/web/frontend/src/components/TraceHound/traceTokens.ts \
        jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx
git commit -m "feat(tracehound): role-named trace theme tokens, migrate panel chrome"
```

---

### Task 2: Replay aggregates per-agent LLM usage (TDD)

**Files:**
- Test: `jiuwenswarm/tests/unit_tests/server/test_tracehound_replay_aggregation.py` (create)
- Modify: `jiuwenswarm/server/agent_ws_server.py` (`_replay_build_turns` usage branches ~10824–10859, `_record_agent` ~10698)
- Modify: `jiuwenswarm/channels/web/frontend/src/stores/traceHoundStore.ts` (`AgentActivity` interface ~line 37)

**Interfaces:**
- Produces: each `agent_activity` entry gains `llm_calls: int`, `tokens: int`, `cost: float` (defaults 0). Frontend `AgentActivity` mirrors: `llm_calls: number; tokens: number; cost: number;`.
- Produces: `hasUsageData(turns)` heuristic used by the `—` display (Task 9): a session has usage data iff `turns.some(t => (t.llm_call_count ?? 0) > 0 || t.total_tokens > 0)`.

- [ ] **Step 1: Write the failing test**

```python
"""TraceHound replay: per-agent LLM usage attribution (team mode)."""
from jiuwenswarm.server.agent_ws_server import AgentWebSocketServer


def _srv() -> AgentWebSocketServer:
    # Skip __init__ (heavy IO setup); _replay_build_turns only uses class attrs.
    return AgentWebSocketServer.__new__(AgentWebSocketServer)


RID = "req_team_turn_1"


def _rec(**kw) -> dict:
    base = {"request_id": RID, "timestamp": 1000.0, "role": "", "event_type": None}
    base.update(kw)
    return base


def test_member_usage_attributed_to_turn_and_agent() -> None:
    records = [
        _rec(role="user", event_type=None, content="plan the trip"),
        _rec(event_type="chat.tool_call", tool_name="fetch_webpage",
             tool_call={"id": "tc1", "name": "fetch_webpage", "arguments": "{}"},
             member_name="prague-foodie"),
        _rec(event_type="chat.tool_result", tool_name="fetch_webpage",
             tool_call_id="tc1", result="success=True", member_name="prague-foodie"),
        _rec(event_type="chat.usage_metadata",
             member_name="prague-foodie",
             metadata={"usage_metadata": {
                 "model_name": "glm-4.7", "input_tokens": 100, "output_tokens": 50,
                 "total_tokens": 150, "cache_tokens": 0, "input_cost": 0.001,
                 "output_cost": 0.002, "total_cost": 0.003, "code": 0, "err_msg": ""},
                 "total_latency_ms": 800.0, "ttft_ms": 300.0, "tpot_ms": 20.0,
                 "result_type": "ok"}),
        _rec(event_type="chat.usage_metadata",
             member_name="prague-foodie",
             metadata={"usage_metadata": {
                 "model_name": "glm-4.7", "input_tokens": 10, "output_tokens": 5,
                 "total_tokens": 15, "cache_tokens": 0, "input_cost": 0.0001,
                 "output_cost": 0.0002, "total_cost": 0.0003, "code": 0, "err_msg": ""},
                 "total_latency_ms": 200.0, "ttft_ms": 100.0, "tpot_ms": 10.0,
                 "result_type": "ok"}),
        _rec(event_type="chat.usage_summary", total_tokens=165,
             member_name="prague-foodie"),
        _rec(event_type="chat.final", role="assistant", content="done"),
    ]
    turns = _srv()._replay_build_turns(records)
    assert len(turns) == 1
    t = turns[0]
    assert t["llm_call_count"] == 2
    assert t["total_tokens"] == 165
    assert abs(t["total_cost"] - 0.0033) < 1e-9
    foodie = next(a for a in t["agent_activity"] if a["name"] == "prague-foodie")
    assert foodie["llm_calls"] == 2
    assert foodie["tokens"] == 165
    assert abs(foodie["cost"] - 0.0033) < 1e-9


def test_single_agent_usage_unattributed_to_agents() -> None:
    records = [
        _rec(role="user", event_type=None, content="hi"),
        _rec(event_type="chat.usage_metadata",
             metadata={"usage_metadata": {
                 "model_name": "glm-4.7", "input_tokens": 1, "output_tokens": 1,
                 "total_tokens": 2, "cache_tokens": 0, "input_cost": 0.0,
                 "output_cost": 0.0, "total_cost": 0.0, "code": 0, "err_msg": ""},
                 "total_latency_ms": 5.0, "ttft_ms": 5.0, "tpot_ms": 5.0,
                 "result_type": "ok"}),
        _rec(event_type="chat.final", role="assistant", content="hello"),
    ]
    turns = _srv()._replay_build_turns(records)
    assert turns[0]["llm_call_count"] == 1
    # Single-agent: no member_name → no agent_activity entries created.
    assert turns[0]["agent_activity"] == []
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd jiuwenswarm && python -m pytest tests/unit_tests/server/test_tracehound_replay_aggregation.py -v`
Expected: FAIL — `KeyError: 'llm_calls'` (agent entries lack the new keys).

- [ ] **Step 3: Implement — extend `_record_agent` + usage branches**

In `_record_agent` (agent_ws_server.py ~10698) add keyword params and counters:

```python
            def _record_agent(
                turn: dict,
                name: str,
                agent_role: str,
                *,
                tool_call: bool = False,
                tool_result: bool = False,
                tool_fail: bool = False,
                response: bool = False,
                llm: bool = False,
                llm_tokens: int = 0,
                llm_cost: float = 0.0,
            ) -> None:
                if not name:
                    return
                acts = turn["_agent_activity"]
                entry = acts.get(name)
                if entry is None:
                    entry = acts[name] = {
                        "name": name,
                        "role": agent_role,
                        "tool_calls": 0,
                        "tool_results": 0,
                        "tool_failures": 0,
                        "responses": 0,
                        "llm_calls": 0,
                        "tokens": 0,
                        "cost": 0.0,
                    }
                if tool_call:
                    entry["tool_calls"] += 1
                if tool_result:
                    entry["tool_results"] += 1
                if tool_fail:
                    entry["tool_failures"] += 1
                if response:
                    entry["responses"] += 1
                if llm:
                    entry["llm_calls"] += 1
                    entry["tokens"] += llm_tokens
                    entry["cost"] += llm_cost
```

In the `chat.usage_metadata` branch (~10824) after `turns[rid]["models_used"].add(model)` append:

```python
                agent_name, agent_role = self._replay_agent_of(rec)
                _record_agent(
                    turns[rid], agent_name, agent_role,
                    llm=True,
                    llm_tokens=um.get("total_tokens", 0) or 0,
                    llm_cost=um.get("total_cost", 0.0) or 0.0,
                )
```

In the `chat.usage_summary` branch (~10854) after `context_window_tokens` append:

```python
                agent_name, agent_role = self._replay_agent_of(rec)
                if agent_name and tokens:
                    acts = turns[rid]["_agent_activity"]
                    entry = acts.get(agent_name)
                    if entry is not None:
                        # usage_summary is cumulative-ish per call; credit deltas are
                        # not reliably derivable — attribute to tokens total once per
                        # summary only when no metadata-based credit happened.
                        if entry["llm_calls"] == 0:
                            entry["tokens"] += tokens
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd jiuwenswarm && python -m pytest tests/unit_tests/server/test_tracehound_replay_aggregation.py -v`
Expected: 2 passed.

- [ ] **Step 5: Frontend type + `—` display groundwork**

In `traceHoundStore.ts` extend `AgentActivity`:

```ts
export interface AgentActivity {
  name: string;
  role: 'leader' | 'member';
  tool_calls: number;
  tool_results: number;
  tool_failures: number;
  responses: number;
  llm_calls: number;
  tokens: number;
  cost: number;
}
```

- [ ] **Step 6: Verify frontend**

Run: `cd jiuwenswarm/channels/web/frontend && npx tsc --noEmit`
Expected: exit 0.

- [ ] **Step 7: Commit**

```bash
git add jiuwenswarm/tests/unit_tests/server/test_tracehound_replay_aggregation.py \
        jiuwenswarm/server/agent_ws_server.py \
        jiuwenswarm/channels/web/frontend/src/stores/traceHoundStore.ts
git commit -m "feat(tracehound): attribute LLM usage per agent in replay aggregation"
```

---

### Task 3: Persist teammate usage events to godview history (write-side)

**Files:**
- Modify: `jiuwenswarm/server/runtime/agent_adapter/interface_deep.py` (member chunk handling; exact site located by spike below)
- Modify (if spike directs): `jiuwenswarm/server/runtime/agent_adapter/team_helpers.py`

**Interfaces:**
- Consumes: `chat.usage_metadata` / `chat.usage_summary` `AgentResponseChunk`s already yielded at `interface_deep.py` ~11571 (`llm_usage` chunk) and ~11941 (summary).
- Produces: history records with `event_type` `chat.usage_metadata`/`chat.usage_summary`, `member_name` set, `request_id` = parent turn's id, appended to the parent godview `history.jsonl`. Task 2's replay then counts them with zero further changes.

- [ ] **Step 1: Spike — locate the teammate mirror write path**

Run: `cd jiuwenswarm && grep -n "member_name" server/runtime/agent_adapter/interface_deep.py | head -20` and inspect the teammate streaming section that stamps `member_name` on `chat.tool_call`/`chat.final` records before `append_history_record(...)`. Also inspect `team_helpers.py:3052` (team `append_history_record` call) and `_build_logical_targets` (~268): teammate LLM output routes `godview + private(member)`.
Identify the exact allowlist/branch that drops `chat.usage_metadata` for teammates (the godview history has tool/final events with `member_name` but zero usage events — verified on the Prague session file).

- [ ] **Step 2: Extend the mirror path to usage events**

Wherever teammate chat events are stamped with `member_name` + parent `request_id` and appended to the parent history, add `chat.usage_metadata` and `chat.usage_summary` to the mirrored event types, wrapped defensively:

```python
                    if et in ("chat.usage_metadata", "chat.usage_summary"):
                        try:
                            append_history_record(
                                parent_session_id,
                                {**payload, "member_name": member_name,
                                 "request_id": parent_request_id},
                            )
                        except Exception:
                            logger.debug("usage mirror failed", exc_info=True)
```

(Adapt names to the actual locals at the site — the invariant is: parent session id, parent request id, `member_name` stamped, exceptions never break the streaming path.)

- [ ] **Step 3: Regression — existing tests**

Run: `cd jiuwenswarm && python -m pytest tests/unit_tests/server tests/unit_tests/server/runtime -q`
Expected: all pass (no existing tests cover the mirror; this guards imports/wiring).

- [ ] **Step 4: Manual acceptance (requires running stack)**

Run a fresh team session (any small 2-member team prompt), then:

```bash
grep -c '"event_type": "chat.usage_metadata"' ~/.jiuwenswarm/agent/sessions/<sid>/history.jsonl
```

Expected: `> 0`, and at least one matching line contains `"member_name"`. Then open its trajectory in the web UI: header shows non-zero LLM calls/tokens, per-agent activity populated.

- [ ] **Step 5: Commit**

```bash
git add jiuwenswarm/server/runtime/agent_adapter/interface_deep.py \
        jiuwenswarm/server/runtime/agent_adapter/team_helpers.py
git commit -m "feat(tracehound): persist teammate LLM usage events to godview history"
```

---

### Task 4: `tracehound.session.mtime` action + `live_updates_enabled` flag

**Files:**
- Modify: `jiuwenswarm/common/schema/message.py` (ReqMethod enum, near `TRACEHOUND_TURNS_LIST` line 368)
- Modify: `jiuwenswarm/server/agent_ws_server.py` (`_handle_replay_request`, new action branch)
- Modify: `jiuwenswarm/gateway/channel_manager/web/app_web_handlers.py` (two allowlists, lines ~716 and ~823)
- Modify: `jiuwenswarm/gateway/channel_manager/tui/tui_connect.py` (`_config_get` payload, ~line 899)
- Modify: `jiuwenswarm/resources/config.yaml` (top-level section)
- Test: `jiuwenswarm/tests/unit_tests/server/test_tracehound_mtime.py` (create)

**Interfaces:**
- Produces: webRequest `tracehound.session.mtime` with params `{session_id}` → `{ok: true, mtime: number | null}` (unix seconds, `null` when no history file).
- Produces: `config.get` payload key `tracehound_live_updates_enabled` (`"true"|"false"`); config.yaml key path `tracehound.live_updates_enabled` (default `false` when absent).

- [ ] **Step 1: Write the failing test**

```python
"""tracehound.session.mtime helper returns history mtime or None."""
import importlib
import sys
from pathlib import Path


def test_mtime_helper(monkeypatch, tmp_path: Path) -> None:
    mod = sys.modules["jiuwenswarm.server.runtime.session.session_history"]
    f = tmp_path / "history.jsonl"
    f.write_text("{}\n", encoding="utf-8")
    monkeypatch.setattr(mod, "get_read_history_path", lambda sid, **kw: f)
    aws = importlib.import_module("jiuwenswarm.server.agent_ws_server")
    srv = aws.AgentWebSocketServer.__new__(aws.AgentWebSocketServer)
    assert srv._tracehound_session_mtime("s1") is not None
    monkeypatch.setattr(mod, "get_read_history_path", lambda sid, **kw: tmp_path / "missing.jsonl")
    assert srv._tracehound_session_mtime("s1") is None
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd jiuwenswarm && python -m pytest tests/unit_tests/server/test_tracehound_mtime.py -v`
Expected: FAIL — `AttributeError: ... no attribute '_tracehound_session_mtime'`.

- [ ] **Step 3: Implement**

`common/schema/message.py` (next to `TRACEHOUND_TURNS_LIST`):

```python
    TRACEHOUND_SESSION_MTIME = "tracehound.session.mtime"
```

`server/agent_ws_server.py` — add a method on `AgentWebSocketServer` (TraceHound helpers section, ~10525):

```python
    @staticmethod
    def _tracehound_session_mtime(session_id: str) -> float | None:
        from jiuwenswarm.server.runtime.session.session_history import get_history_mtime
        try:
            return get_history_mtime(session_id)
        except Exception:
            return None
```

In `_handle_replay_request` (~10982) add before `turns_list`:

```python
        if action == "session_mtime":
            mtime = self._tracehound_session_mtime(session_id)
            await self._send_response(ws, request, {"ok": True, "mtime": mtime}, send_lock)  # adapt to the file's actual send helper
            return
```

(Adapt the send call to match how the neighboring `turns_list`/`turn_get` branches send their payload.)

`app_web_handlers.py`: add `"tracehound.session.mtime",` next to both `"tracehound.turns.list",` entries (~716, ~823).

`tui_connect.py` `_config_get` (~after `auto_recap_enabled` block, ~line 894):

```python
            _trace_cfg = raw.get("tracehound") or {}
            payload["tracehound_live_updates_enabled"] = (
                "true" if _trace_cfg.get("live_updates_enabled", False) else "false"
            )
```

`resources/config.yaml` (top level, near `agent_observability:`):

```yaml
tracehound:
  # Live trajectory auto-refresh in the web chat panel (mtime-gated polling).
  live_updates_enabled: false
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd jiuwenswarm && python -m pytest tests/unit_tests/server/test_tracehound_mtime.py tests/unit_tests/server/test_tracehound_replay_aggregation.py -v`
Expected: all pass.

- [ ] **Step 5: Commit**

```bash
git add jiuwenswarm/common/schema/message.py \
        jiuwenswarm/server/agent_ws_server.py \
        jiuwenswarm/gateway/channel_manager/web/app_web_handlers.py \
        jiuwenswarm/gateway/channel_manager/tui/tui_connect.py \
        jiuwenswarm/resources/config.yaml \
        jiuwenswarm/tests/unit_tests/server/test_tracehound_mtime.py
git commit -m "feat(tracehound): session mtime action + live_updates_enabled config flag"
```

---

### Task 5: `refreshTurns()` + live polling (frontend, flag-gated)

**Files:**
- Create: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/traceLive.ts`
- Create: `jiuwenswarm/channels/web/frontend/tests/traceLive.test.mjs`
- Modify: `jiuwenswarm/channels/web/frontend/package.json` (scripts)
- Modify: `jiuwenswarm/channels/web/frontend/src/stores/traceHoundStore.ts`
- Modify: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx` (TrajectoryPanel)
- Modify: `jiuwenswarm/channels/web/frontend/src/App.tsx`

**Interfaces:**
- Consumes: Task 4's `tracehound.session.mtime` (`{mtime: number | null}`).
- Produces: `traceLive.ts` exports `shouldRefetch(prevMtime: number | null, nextMtime: number | null): boolean` and `POLL_INTERVAL_MS = 5000`.
- Produces: store action `refreshTurns(sessionId: string): Promise<void>` — like `selectSession` but preserves `selectedTurnId`/`turnRecords` (re-fetches records when the selected turn's `event_count` grew); sets `turns`/`sessionStats` in place.
- Produces: `TrajectoryPanel` prop `liveUpdatesEnabled?: boolean`; App.tsx derives it from `serverConfig.tracehound_live_updates_enabled === 'true'`.

- [ ] **Step 1: Write the failing test (`tests/traceLive.test.mjs`)**

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { shouldRefetch, POLL_INTERVAL_MS } from '../src/components/TraceHound/traceLive.ts';

test('refetch only when mtime actually changes', () => {
  assert.equal(shouldRefetch(null, 123.5), true);   // first observation of an existing file
  assert.equal(shouldRefetch(123.5, 123.5), false); // unchanged
  assert.equal(shouldRefetch(123.5, 124.0), true);  // changed
  assert.equal(shouldRefetch(123.5, null), false);  // file vanished — do not hammer
  assert.equal(shouldRefetch(null, null), false);
});

test('poll interval is 5s', () => {
  assert.equal(POLL_INTERVAL_MS, 5000);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd jiuwenswarm/channels/web/frontend && npm run test:trace-live`
Expected: FAIL (script/module missing). Add the script first if needed:

```json
    "test:trace-live": "tsc src/components/TraceHound/traceLive.ts --target ES2020 --module ES2020 --moduleResolution Bundler --rootDir src --outDir node_modules/.cache/trace-live --skipLibCheck --noEmitOnError && node --test tests/traceLive.test.mjs",
```

- [ ] **Step 3: Implement `traceLive.ts`**

```ts
/** Live-update decision logic for the trajectory panel (pure, tested). */
export const POLL_INTERVAL_MS = 5000;

/** Decide whether a new mtime warrants re-fetching turns. */
export function shouldRefetch(prevMtime: number | null, nextMtime: number | null): boolean {
  if (nextMtime === null) return false;      // no history file — never poll-fetch
  if (prevMtime === null) return true;       // first sight of an existing file
  return nextMtime > prevMtime;              // only forward changes
}
```

Compile artifact import path in the test: adjust the test import to `'../node_modules/.cache/trace-live/traceLive.js'` if the repo's other tests do the same (follow `test:frontend-platform` conventions).

- [ ] **Step 4: Run test to verify it passes**

Run: `npm run test:trace-live`
Expected: 2 passing.

- [ ] **Step 5: Store `refreshTurns`**

In `traceHoundStore.ts` (after `openCurrentSession`):

```ts
  refreshTurns: async (sessionId) => {
    const prevEventCount = new Map(get().turns.map(t => [t.turn_id, t.event_count]));
    try {
      const res = await webRequest<{ ok: boolean; turns: TurnSummary[]; session_stats: SessionStats }>(
        'tracehound.turns.list', { session_id: sessionId }
      );
      const turns = Array.isArray(res?.turns) ? res.turns : [];
      const selectedTurnId = get().selectedTurnId;
      const selected = selectedTurnId ? turns.find(t => t.turn_id === selectedTurnId) : undefined;
      const selectedGrew = selectedTurnId &&
        (selected?.event_count ?? 0) > (prevEventCount.get(selectedTurnId) ?? 0);
      set({
        turns,
        sessionStats: res?.session_stats ?? get().sessionStats,
        ...(selectedGrew ? { turnRecords: [] } : {}),
      });
      if (selectedGrew && selectedTurnId) {
        await get().loadTurnRecords?.(selectedTurnId); // reuse existing record loader if present; otherwise omit
      }
    } catch {
      /* transient — next tick retries */
    }
  },
```

(Declare `refreshTurns: (sessionId: string) => Promise<void>;` in the store interface. If no `loadTurnRecords` action exists, inline the existing `turn_get` call used by `selectTurn`. `turnRecords: []` clearing + reload keeps the detail view honest while preserving scroll/selection for the list.)

- [ ] **Step 6: Poll in `TrajectoryPanel`**

Extend props with `liveUpdatesEnabled?: boolean`. Inside the component:

```tsx
  const [live, setLive] = useState(false);
  const mtimeRef = useRef<number | null>(null);
  useEffect(() => {
    if (!liveUpdatesEnabled || !isConnected) return;
    let stopped = false;
    const tick = async () => {
      try {
        const r = await webRequest<{ mtime: number | null }>('tracehound.session.mtime', { session_id: sessionId });
        if (stopped) return;
        const next = r?.mtime ?? null;
        if (shouldRefetch(mtimeRef.current, next)) {
          await useTraceHoundStore.getState().refreshTurns(sessionId);
        }
        mtimeRef.current = next;
        setLive(true);
      } catch { setLive(false); }
    };
    void tick();
    const id = window.setInterval(tick, POLL_INTERVAL_MS);
    return () => { stopped = true; window.clearInterval(id); setLive(false); };
  }, [sessionId, liveUpdatesEnabled, isConnected]);
```

Pause while analyzing: add `analyzing` from the store to the effect deps and early-return when true. Render the pulse in the header when `live`:

```tsx
  {live && <span style={{ fontSize: 10, color: C.ok }}>● LIVE</span>}
```

Import `shouldRefetch, POLL_INTERVAL_MS` from `./traceLive` and `webRequest` following the store's existing import pattern. In `App.tsx`, pass
`liveUpdatesEnabled={serverConfig?.tracehound_live_updates_enabled === 'true'}` (match `serverConfig`'s actual type — it is a `Record<string, unknown>`-like state; coerce with `String(...)`).

- [ ] **Step 7: Verify + commit**

Run: `cd jiuwenswarm/channels/web/frontend && npx tsc --noEmit && npm run build && npm run test:trace-live`
Expected: all green.

```bash
git add jiuwenswarm/channels/web/frontend/src/components/TraceHound/traceLive.ts \
        jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx \
        jiuwenswarm/channels/web/frontend/src/stores/traceHoundStore.ts \
        jiuwenswarm/channels/web/frontend/src/App.tsx \
        jiuwenswarm/channels/web/frontend/tests/traceLive.test.mjs \
        jiuwenswarm/channels/web/frontend/package.json
git commit -m "feat(tracehound): mtime-gated live trajectory refresh behind config flag"
```

---

### Task 6: Highlights strip (free, no LLM)

**Files:**
- Create: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/highlights.ts`
- Create: `jiuwenswarm/channels/web/frontend/tests/traceHighlights.test.mjs`
- Modify: `jiuwenswarm/channels/web/frontend/package.json` (script `test:trace-highlights`)
- Modify: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx` (render strip in `TurnListView` above Diagnosis)

**Interfaces:**
- Consumes: `TurnSummary` from the store.
- Produces: `buildHighlights(turns: TurnSummary[]): Highlight[]` where

```ts
export interface Highlight {
  id: string;            // stable key
  icon: string;          // emoji
  kind: 'retries' | 'toolFailures' | 'slowest' | 'context' | 'problems';
  label: string;         // short text (i18n keys applied at render time in Task 9)
  turnIds: string[];     // turns to jump/filter to
}
```

- [ ] **Step 1: Write the failing test**

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { buildHighlights } from '../node_modules/.cache/trace-highlights/highlights.js';

const turn = (over) => ({
  turn_id: 't1', turn_index: 0, user_content: 'q', tool_names: [], skill_names: [],
  has_final: true, has_error: false, error_category: null, total_tokens: 0,
  tool_failures: 0, file_count: 0, final_length: 0, duration_seconds: 5,
  retry_count: 1, was_deferred: false, query_type: 'general',
  outcome: 'completed', issues: [], mode: null, llm_call_count: 0, event_count: 1,
  ...over,
});

test('empty/healthy sessions produce no highlights', () => {
  assert.deepEqual(buildHighlights([turn({})]), []);
});

test('retries, failures, slow, problems surface', () => {
  const hs = buildHighlights([
    turn({ turn_id: 'a', retry_count: 45, outcome: 'completed_with_issues' }),
    turn({ turn_id: 'b', tool_failures: 3, duration_seconds: 1080 }),
  ]);
  const kinds = hs.map(h => h.kind).sort();
  assert.deepEqual(kinds, ['problems', 'retries', 'slowest', 'toolFailures']);
  const retries = hs.find(h => h.kind === 'retries');
  assert.equal(retries.turnIds[0], 'a');
});
```

- [ ] **Step 2: Run to verify it fails** — `npm run test:trace-highlights` (add script like Task 5's). Expected: module missing.

- [ ] **Step 3: Implement `highlights.ts`**

```ts
import type { TurnSummary } from '../../stores/traceHoundStore';

export interface Highlight {
  id: string;
  icon: string;
  kind: 'retries' | 'toolFailures' | 'slowest' | 'context' | 'problems';
  label: string;
  turnIds: string[];
}

export function buildHighlights(turns: TurnSummary[]): Highlight[] {
  const out: Highlight[] = [];
  if (turns.length === 0) return out;

  const topRetry = [...turns].sort((a, b) => b.retry_count - a.retry_count)[0];
  if (topRetry.retry_count > 1) {
    out.push({ id: 'retries', icon: '🔁', kind: 'retries',
      label: `#${topRetry.turn_index + 1} — ${topRetry.retry_count} attempts`, turnIds: [topRetry.turn_id] });
  }

  const totalFail = turns.reduce((s, t) => s + (t.tool_failures ?? 0), 0);
  if (totalFail > 0) {
    const ids = turns.filter(t => (t.tool_failures ?? 0) > 0).map(t => t.turn_id);
    out.push({ id: 'toolFailures', icon: '✗', kind: 'toolFailures', label: `${totalFail} tool failures`, turnIds: ids });
  }

  const timed = turns.filter(t => t.retry_count <= 1 && t.duration_seconds > 0);
  const slowest = [...timed].sort((a, b) => b.duration_seconds - a.duration_seconds)[0];
  if (slowest && slowest.duration_seconds > 60) {
    out.push({ id: 'slowest', icon: '⏱', kind: 'slowest',
      label: `#${slowest.turn_index + 1} took ${Math.round(slowest.duration_seconds)}s`, turnIds: [slowest.turn_id] });
  }

  const maxCtx = Math.max(...turns.map(t => t.context_usage_percent ?? 0), 0);
  if (maxCtx > 80) {
    const ids = turns.filter(t => (t.context_usage_percent ?? 0) > 80).map(t => t.turn_id);
    out.push({ id: 'context', icon: '📏', kind: 'context', label: `context ${maxCtx.toFixed(0)}%`, turnIds: ids });
  }

  const withProblems = turns.filter(t => t.outcome !== 'completed' && t.outcome !== 'deferred');
  if (withProblems.length > 0 && turns.length > 1) {
    out.push({ id: 'problems', icon: '⚠', kind: 'problems',
      label: `${withProblems.length} of ${turns.length} with problems`, turnIds: withProblems.map(t => t.turn_id) });
  }

  return out.slice(0, 4);
}
```

- [ ] **Step 4: Run to verify it passes** — `npm run test:trace-highlights`. Expected: 2 passing.

- [ ] **Step 5: Render the strip in `TurnListView`**

Above the Diagnosis block:

```tsx
{!loading && (() => { const hs = buildHighlights(turns); return hs.length > 0 ? (
  <div data-testid="tracehound-highlights" style={{ display: 'flex', gap: 8, flexWrap: 'wrap', marginBottom: 16 }}>
    {hs.map(h => (
      <button key={h.id} style={{
        display: 'flex', alignItems: 'center', gap: 6, fontSize: 12, cursor: 'pointer',
        padding: '5px 10px', borderRadius: 8, border: `1px solid ${C.border}`,
        background: C.surface, color: C.textMuted,
      }} onClick={() => isConnected && selectTurn(h.turnIds[0])} title={h.label}>
        <span>{h.icon}</span><span>{h.label}</span>
      </button>
    ))}
  </div>
) : null; })()}
```

- [ ] **Step 6: Verify + commit**

Run: `cd jiuwenswarm/channels/web/frontend && npx tsc --noEmit && npm run build && npm run test:trace-highlights`

```bash
git add jiuwenswarm/channels/web/frontend/src/components/TraceHound/highlights.ts \
        jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx \
        jiuwenswarm/channels/web/frontend/tests/traceHighlights.test.mjs \
        jiuwenswarm/channels/web/frontend/package.json
git commit -m "feat(tracehound): free highlights strip of top session signals"
```

---

### Task 7: Timeline SVG + per-agent activity charts

**Files:**
- Create: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/traceCharts.tsx`
- Modify: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx` (AnalyticsPanel additions)

**Interfaces:**
- Consumes: `TurnSummary` (incl. `agent_activity` with Task 2 fields), `HistoryRecord` for detail-timeline, `C`/`cat` tokens.
- Produces (exported from `traceCharts.tsx`, reused by Task 10):
  - `<TimelineBand records={HistoryRecord[]} height={number} onClickRecord?(r) />` — horizontal wall-clock strip.
  - `<PerAgentCard turn={TurnSummary} onAgentClick?(name) />` — grouped bars per member.
  - `EVENT_COLORS: Record<string, string>` mapping event_type → token color.

- [ ] **Step 1: Implement `traceCharts.tsx`**

```tsx
import type { HistoryRecord, TurnSummary } from '../../stores/traceHoundStore';
import { C, cat } from './traceTokens';

export const EVENT_COLORS: Record<string, string> = {
  user: C.info,
  'chat.reasoning': C.violet,
  'chat.tool_call': C.warn,
  'chat.tool_result': C.ok,
  'chat.final': C.info,
  'chat.file': C.teal,
  'chat.usage_metadata': C.violet,
  'chat.error': C.danger,
};

/** Horizontal wall-clock strip of one turn's records (SVG, token-colored). */
export function TimelineBand({ records, height = 44, onClickRecord }: {
  records: HistoryRecord[]; height?: number; onClickRecord?: (r: HistoryRecord) => void;
}) {
  const pts = records.filter(r => (r.timestamp ?? 0) > 0);
  if (pts.length === 0) return null;
  const t0 = Math.min(...pts.map(r => r.timestamp!));
  const t1 = Math.max(...pts.map(r => r.timestamp!));
  const span = Math.max(t1 - t0, 0.001);
  const W = 600;
  const cy = height / 2;
  return (
    <svg viewBox={`0 0 ${W} ${height}`} style={{ width: '100%', height }} role="img">
      <line x1={0} y1={cy} x2={W} y2={cy} stroke={C.border} strokeWidth={1} />
      {pts.map((r, i) => {
        const x = ((r.timestamp! - t0) / span) * (W - 8) + 4;
        const et = r.role === 'user' ? 'user' : (r.event_type ?? '');
        const color = EVENT_COLORS[et] ?? C.textFaint;
        const failed = r.event_type === 'chat.tool_result' &&
          (r.result ?? '').includes('success=False');
        return (
          <circle key={i} cx={x} cy={cy} r={failed ? 5 : 3.5}
            fill={failed ? C.danger : color} opacity={0.9} style={{ cursor: onClickRecord ? 'pointer' : 'default' }}
            onClick={() => onClickRecord?.(r)}>
            <title>{`${et} @ +${((r.timestamp! - t0)).toFixed(1)}s${failed ? ' (failed)' : ''}`}</title>
          </circle>
        );
      })}
    </svg>
  );
}

/** Team-mode per-agent activity bars (tool calls / failures / LLM calls / tokens). */
export function PerAgentCard({ turn, onAgentClick }: {
  turn: TurnSummary; onAgentClick?: (name: string) => void;
}) {
  const acts = turn.agent_activity ?? [];
  if (acts.length === 0) return null;
  const maxVal = Math.max(...acts.flatMap(a => [a.tool_calls, a.tool_failures, a.llm_calls, Math.ceil(a.tokens / 1000)]), 1);
  return (
    <div data-testid="tracehound-per-agent" style={{ background: C.surface, borderRadius: 6, padding: 12, border: `1px solid ${C.border}` }}>
      <div style={{ fontSize: 11, fontWeight: 700, color: C.text, marginBottom: 8 }}>Per-agent activity</div>
      {acts.map((a, i) => (
        <div key={a.name} style={{ marginBottom: 8, cursor: onAgentClick ? 'pointer' : 'default' }}
             onClick={() => onAgentClick?.(a.name)}>
          <div style={{ display: 'flex', justifyContent: 'space-between', fontSize: 11, marginBottom: 3 }}>
            <span style={{ color: cat(i + 1) }}>{a.role === 'leader' ? '⛨ ' : ''}{a.name}</span>
            <span style={{ color: C.textMuted }}>
              {a.llm_calls > 0 ? `${a.llm_calls} llm · ${a.tokens.toLocaleString()} tok` : '—'}
            </span>
          </div>
          {([['tools', a.tool_calls, C.warn], ['fails', a.tool_failures, C.danger], ['llm', a.llm_calls, C.violet]] as const)
            .map(([label, v, color]) => (
              <div key={label} style={{ display: 'flex', alignItems: 'center', gap: 6, marginBottom: 2 }}>
                <span style={{ fontSize: 9, color: C.textFaint, width: 32 }}>{label}</span>
                <div style={{ flex: 1, height: 4, background: C.surfaceMuted, borderRadius: 2 }}>
                  <div style={{ height: 4, width: `${(v / maxVal) * 100}%`, background: color, borderRadius: 2 }} />
                </div>
                <span style={{ fontSize: 9, color: C.textMuted, width: 24, textAlign: 'right' }}>{v}</span>
              </div>
            ))}
        </div>
      ))}
    </div>
  );
}
```

- [ ] **Step 2: Wire into `AnalyticsPanel`**

Inside the grid (after the Tool usage card), when the session's turns have `agent_activity`:

```tsx
{(() => {
  const teamTurns = turns.filter(t => (t.agent_activity?.length ?? 0) > 0);
  return teamTurns.length > 0 ? <PerAgentCard turn={teamTurns[0]} /> : null;
})()}
```

(Aggregated across turns: sum per agent name in a follow-up loop if `teamTurns.length > 1` — merge entries by name before rendering.)

Also add a per-turn timeline above the user-message list rows: inside the turn card render `<TimelineBand records={/* turn-level proxy */ []} />` — the list view lacks records; render `TimelineBand` only in `TurnDetailView` (top of the records stack) where `turnRecords` exists. Remove the list-view placeholder.

- [ ] **Step 3: Render `TimelineBand` in `TurnDetailView`**

At the top of the detail stack (after the back row): `<TimelineBand records={turnRecords} onClickRecord={r => document.getElementById(`rec-${r.id}`)?.scrollIntoView({ behavior: 'smooth', block: 'center' })} />` and add `id={`rec-${rec.id}`}` to the `RecordCard` wrapper div in the display list.

- [ ] **Step 4: Verify + commit**

Run: `cd jiuwenswarm/channels/web/frontend && npx tsc --noEmit && npm run build`
Expected: green.

```bash
git add jiuwenswarm/channels/web/frontend/src/components/TraceHound/traceCharts.tsx \
        jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx
git commit -m "feat(tracehound): SVG timeline band + per-agent activity charts"
```

---

### Task 8: Cross-links (Stats → turn detail at failing record)

**Files:**
- Modify: `jiuwenswarm/channels/web/frontend/src/stores/traceHoundStore.ts`
- Modify: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx` (tool-usage rows + TurnDetailView scroll)

**Interfaces:**
- Produces: store state `focusRecordId: string | null` + action `jumpToTurn(turnId: string, recordId?: string): void` (sets `selectedTurnId`, `focusRecordId`; `selectTurn` clears `focusRecordId`).
- Consumes: `tool_results_detail` on `TurnSummary` (existing) to resolve failing records.

- [ ] **Step 1: Store additions**

Interface: `focusRecordId: string | null;` and `jumpToTurn: (turnId: string, recordId?: string) => void;`. Implementation:

```ts
  jumpToTurn: (turnId, recordId) => {
    set({ selectedTurnId: turnId, focusRecordId: recordId ?? null });
    void get().loadTurnDetail(turnId); // reuse the same fetch selectTurn performs
  },
```

(Reuse `selectTurn`'s internal fetch — if it is inline in `selectTurn`, extract or duplicate the `turn_get` call; `selectTurn` must also reset `focusRecordId: null`.)

- [ ] **Step 2: Tool-usage rows become buttons**

In `AnalyticsPanel`'s tool usage map, wrap the row in an `onClick` that resolves the first failing record:

```tsx
  const firstFail = turns
    .flatMap(t => (t.tool_results_detail ?? []).map(r => ({ turnId: t.turn_id, r })))
    .find(x => x.r.tool_name === tool && (x.r.failed || (x.r.result ?? '').includes('success=False')));
  // row onClick: u.failed > 0 && firstFail && isConnected && jumpToTurn(firstFail.turnId, firstFail.r.tool_call_id)
```

Add `cursor: u.failed > 0 ? 'pointer' : 'default'` and a `title="Jump to first failure"` on failing rows. Thread `jumpToTurn` into `AnalyticsPanel` props (or read from the store hook directly — `AnalyticsPanel` already receives only `turns`; prefer reading the store inside).

- [ ] **Step 3: TurnDetailView consumes `focusRecordId`**

After `turnRecords` load, scroll to the record whose `tool_call_id` matches:

```tsx
  const focusRecordId = useTraceHoundStore(s => s.focusRecordId);
  useEffect(() => {
    if (!focusRecordId || turnRecords.length === 0) return;
    const el = document.getElementById(`rec-${focusRecordId}`) ??
      document.querySelector(`[data-tool-call-id="${focusRecordId}"]`);
    el?.scrollIntoView({ behavior: 'smooth', block: 'center' });
  }, [focusRecordId, turnRecords]);
```

Add `data-tool-call-id={rec.tool_call_id}` to `RecordCard`'s wrapper for tool records (ids on `rec-…` remain the primary anchor from Task 7).

- [ ] **Step 4: Verify + commit**

Run: `cd jiuwenswarm/channels/web/frontend && npx tsc --noEmit && npm run build`

```bash
git add jiuwenswarm/channels/web/frontend/src/stores/traceHoundStore.ts \
        jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx
git commit -m "feat(tracehound): cross-link failing tools to turn detail records"
```

---

### Task 9: i18n (en/zh) + full token migration + dead-key cleanup

**Files:**
- Modify: `jiuwenswarm/channels/web/frontend/src/i18n/locales/en.json`, `zh.json`
- Modify: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx` (all strings + remaining hex)
- Modify: `jiuwenswarm/channels/web/frontend/src/utils/frontendPlatform.ts` (drop `'tracehound'` from `MainNavKey`)

**Interfaces:**
- Produces: `traceHound.*` key family (extends existing `traceHound.trajectoryPanel`). `—` rendering for absent usage via `hasUsageData` (Task 2 heuristic).

- [ ] **Step 1: Add the key families**

Under `"traceHound"` in both locales add (en shown; zh gets natural translations):

```json
{
  "trajectoryPanel": "Trajectory",
  "live": "LIVE",
  "close": "Close",
  "diagnosis": { "title": "Diagnosis", "none": "No diagnosis yet", "disclaimer": "uses LLM (costs tokens, takes time)", "diagnose": "Diagnose", "rerun": "Re-run", "running": "Running LLM diagnosis…", "stale": "stale", "healthy": "No issues found. Session looks healthy!" },
  "stats": { "title": "Stats", "fromData": "from session data", "completed": "Completed", "withProblems": "With problems", "noResponse": "No response", "errors": "Errors", "deferred": "Deferred", "retries": "Total retries", "toolFailures": "Tool failures", "perUserMessage": "Per user message", "outcome": "Outcome", "tokens": "Tokens", "llmCalls": "LLM calls", "duration": "Duration", "toolsPerMsg": "Tools per msg", "skillsPerMsg": "Skills per msg", "errorCategories": "Error categories", "queryTypes": "Query types", "toolUsage": "Tool usage", "skillUsage": "Skill usage", "userMsgs": "user msgs", "noData": "—" },
  "messages": { "title": "User messages", "log": "one-by-one log", "shown": "{{shown}} shown · {{total}} total", "filtered": "(filtered)", "withProblems": "With Problems", "noResponse": "No Response", "errors": "Errors", "deferred": "Deferred", "withRetries": "With Retries", "slow": "Slow", "reset": "Reset", "noMessages": "No user messages found for this session.", "noMatch": "No user messages match the active filters.", "attempts": "attempts", "failed": "failed" },
  "highlights": { "title": "Highlights", "toolFailures": "{{count}} tool failures", "retries": "#{{index}} — {{count}} attempts", "slowest": "#{{index}} took {{seconds}}s", "context": "context {{percent}}%", "problems": "{{count}} of {{total}} with problems" },
  "graph": { "records": "Records", "graph": "Graph", "aggregated": "Aggregated", "expanded": "Expanded", "runs": "runs" },
  "perAgent": { "title": "Per-agent activity", "tools": "tools", "fails": "fails", "llm": "llm", "noUsage": "—" }
}
```

Then sweep `index.tsx`: replace every user-visible literal with `t('traceHound.…')` following the app's existing `useTranslation`/`t` import pattern (same as `traceHound.trajectoryPanel` usage in `ChatPanel`). Interpolated counts use the `{{var}}` pattern above.

- [ ] **Step 2: Finish hex→token migration**

Replace every remaining hardcoded color in `index.tsx` with `C.*` per this mapping (complete list):

| Hex | Token |
|---|---|
| `#ef4444 #dc2626 #fca5a5 #fee2e2 #fecaca #fff5f5` | `C.danger` / `C.dangerSubtle` |
| `#f59e0b #d97706 #fdba74 #ffedd5 #fde68a #fffbeb #fef3c7 #b45309` | `C.warn` / `C.warnSubtle` |
| `#10b981 #059669 #a7f3d0 #ecfdf5 #16a34a #15803d #14532d #bbf7d0` | `C.ok` / `C.okSubtle` |
| `#6366f1 #8b5cf6 #7c3aed #4c1d95 #eef2ff #ede9fe #ddd6fe #c4b5fd #a5b4fc #bfdbfe` | `C.violet` / `C.violetSubtle` |
| `#3b82f6 #1d4ed8 #93c5fd #dbeafe #e0f2fe #0369a1 #06b6d4` | `C.info` / `C.teal` |
| `#111827 #374151 #1f2937` | `C.text` |
| `#6b7280 #4b5563` | `C.textMuted` |
| `#9ca3af` | `C.textFaint` |
| `#e5e7eb #f3f4f6 #f9fafb #fafafa #d1d5db #cbd5e1 #94a3b8` | `C.border` / `C.surfaceMuted` / `C.borderStrong` / `C.textFaint` |
| `#fff / '#fff'` (backgrounds) | `C.surface` |

- [ ] **Step 3: `—` for absent usage**

Compute `const hasUsage = turns.some(t => (t.llm_call_count ?? 0) > 0 || t.total_tokens > 0);` once in `TurnListView`; when false, render the header's LLM-calls/tokens chips and the Stats tokens/LLM rows as `t('traceHound.stats.noData')` (`—`) instead of `0`.

- [ ] **Step 4: Dead-key cleanup**

Remove `"tracehound": "TraceHound"` from the `nav` block in **both** locales; in `frontendPlatform.ts` remove `| 'tracehound'` from the `MainNavKey` union (grep first: `grep -rn "'tracehound'" src` must show zero non-i18n usages after removal).

- [ ] **Step 5: Verify + commit**

Run: `cd jiuwenswarm/channels/web/frontend && npx tsc --noEmit && npm run build`
Playwright spot-check both locales (switch app language) — every panel string localized, old team session shows `—`.

```bash
git add jiuwenswarm/channels/web/frontend/src/i18n/locales/en.json \
        jiuwenswarm/channels/web/frontend/src/i18n/locales/zh.json \
        jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx \
        jiuwenswarm/channels/web/frontend/src/utils/frontendPlatform.ts
git commit -m "feat(tracehound): full en/zh i18n, token migration, graceful no-usage display"
```

---

### Task 10: Langfuse-style agent graph (`Records | Graph` tabs)

**Files:**
- Create: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/traceGraph.ts` (pure model builder)
- Create: `jiuwenswarm/channels/web/frontend/tests/traceGraph.test.mjs` + script `test:trace-graph`
- Create: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/TraceGraph.tsx` (renderer)
- Modify: `jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx` (tabs in `TurnDetailView`)

**Interfaces:**
- Consumes: `HistoryRecord[]` (`turnRecords`), `C`/`cat` tokens, `EVENT_COLORS` from Task 7.
- Produces: `traceGraph.ts` exports:

```ts
export type GraphNode = {
  id: string; label: string; kind: 'user' | 'agent' | 'tool' | 'llm' | 'final';
  count: number; agent?: string; recordIds: string[];
};
export type GraphEdge = { from: string; to: string; kind: 'seq' | 'pair' | 'spawn' | 'cycle' };
export function buildGraph(records: HistoryRecord[], mode: 'aggregated' | 'expanded'): {
  nodes: GraphNode[]; edges: GraphEdge[];
};
```

- [ ] **Step 1: Write the failing test**

```js
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { buildGraph } from '../node_modules/.cache/trace-graph/traceGraph.js';

const rec = (over) => ({ id: 'r0', role: 'user', request_id: 'q', content: '', timestamp: 1, event_type: null, ...over });
const records = [
  rec({ id: 'u1', role: 'user', content: 'go', timestamp: 1 }),
  rec({ id: 'm1', event_type: 'chat.usage_metadata', timestamp: 2, member_name: 'foodie',
       metadata: { usage_metadata: { total_tokens: 5, total_cost: 0, model_name: 'm' } } }),
  rec({ id: 't1', event_type: 'chat.tool_call', timestamp: 3, tool_name: 'search',
       tool_call: { id: 'tc1', name: 'search', arguments: '{}' }, member_name: 'foodie' }),
  rec({ id: 'r1', event_type: 'chat.tool_result', timestamp: 4, tool_name: 'search',
       tool_call_id: 'tc1', result: 'ok', member_name: 'foodie' }),
  rec({ id: 't2', event_type: 'chat.tool_call', timestamp: 5, tool_name: 'search',
       tool_call: { id: 'tc2', name: 'search', arguments: '{}' }, member_name: 'foodie' }),
  rec({ id: 'r2', event_type: 'chat.tool_result', timestamp: 6, tool_name: 'search',
       tool_call_id: 'tc2', result: 'ok', member_name: 'foodie' }),
  rec({ id: 'f1', event_type: 'chat.final', role: 'assistant', content: 'done', timestamp: 7 }),
];

test('expanded: one node per call, paired edges', () => {
  const g = buildGraph(records, 'expanded');
  const tools = g.nodes.filter(n => n.kind === 'tool');
  assert.equal(tools.length, 2);
  assert.ok(g.edges.some(e => e.kind === 'pair'));          // tc call→result
  assert.ok(g.edges.every(e => e.kind !== 'cycle'));
});

test('aggregated: same-name tools collapse with counter + cycle', () => {
  const g = buildGraph(records, 'aggregated');
  const tools = g.nodes.filter(n => n.kind === 'tool' && n.label === 'search');
  assert.equal(tools.length, 1);
  assert.equal(tools[0].count, 2);
  assert.ok(g.edges.some(e => e.kind === 'cycle'));
});
```

- [ ] **Step 2: Run to verify it fails** — `npm run test:trace-graph` (add script). Expected: module missing.

- [ ] **Step 3: Implement `traceGraph.ts`**

```ts
import type { HistoryRecord } from '../../stores/traceHoundStore';

export type GraphNode = {
  id: string; label: string; kind: 'user' | 'agent' | 'tool' | 'llm' | 'final';
  count: number; agent?: string; recordIds: string[];
};
export type GraphEdge = { from: string; to: string; kind: 'seq' | 'pair' | 'spawn' | 'cycle' };
export type GraphMode = 'aggregated' | 'expanded';

/** Build the agent-workflow graph for one turn from its history records.
 *  Edges are inferred: temporal sequence, tool_call_id pairing, spawn links. */
export function buildGraph(records: HistoryRecord[], mode: GraphMode): { nodes: GraphNode[]; edges: GraphEdge[] } {
  const nodes = new Map<string, GraphNode>();
  const edges: GraphEdge[] = [];
  const seq: string[] = []; // ordered expanded node ids

  const add = (n: GraphNode) => { nodes.set(n.id, n); };
  const keyOf = (kind: GraphNode['kind'], label: string, agent?: string) =>
    mode === 'aggregated' ? `${kind}:${agent ?? ''}:${label}` : `${kind}:${agent ?? ''}:${label}:${seq.length}`;

  const agentOf = (r: HistoryRecord) => (r.member_name ?? '').trim() || undefined;

  for (const r of records) {
    const et = r.role === 'user' ? 'user' : (r.event_type ?? '');
    const agent = agentOf(r);
    if (et === 'user') {
      add({ id: 'user', label: 'user', kind: 'user', count: 1, recordIds: [r.id] });
      seq.push('user');
    } else if (et === 'chat.tool_call') {
      const label = r.tool_call?.name || r.tool_name || 'tool';
      const id = keyOf('tool', label, agent);
      const ex = nodes.get(id);
      if (ex) { ex.count += 1; ex.recordIds.push(r.id); }
      else add({ id, label, kind: 'tool', count: 1, agent, recordIds: [r.id] });
      seq.push(id);
    } else if (et === 'chat.usage_metadata') {
      const id = keyOf('llm', 'llm', agent);
      const ex = nodes.get(id);
      if (ex) { ex.count += 1; ex.recordIds.push(r.id); }
      else add({ id, label: 'llm', kind: 'llm', count: 1, agent, recordIds: [r.id] });
      seq.push(id);
    } else if (et === 'chat.final') {
      if ((r.content ?? '').trim()) {
        const id = keyOf('final', 'final', agent);
        const ex = nodes.get(id);
        if (ex) { ex.count += 1; ex.recordIds.push(r.id); }
        else add({ id, label: 'final', kind: 'final', count: 1, agent, recordIds: [r.id] });
        seq.push(id);
      }
    } else if (et === 'chat.tool_result' || et === 'chat.tool_update' || et === 'chat.reasoning' || et === 'chat.error') {
      // not nodes; tool_result participates via pairing below
    }
  }

  // Sequential edges (dedup in aggregated mode)
  const seen = new Set<string>();
  for (let i = 1; i < seq.length; i++) {
    const a = seq[i - 1], b = seq[i];
    if (a === b) continue;
    const k = `${a}->${b}`;
    if (mode === 'aggregated') {
      if (nodes.get(a) && nodes.get(b) && !nodes.get(a)!.recordIds.includes('') ) {
        if (seen.has(k)) { edges.push({ from: a, to: b, kind: 'cycle' }); continue; }
      }
    }
    if (!seen.has(k)) { seen.add(k); edges.push({ from: a, to: b, kind: 'seq' }); }
  }

  // Pairing: tool_call -> its tool_result position (call node already in seq order;
  // the pair edge connects a tool node to the next node after its result).
  for (const r of records) {
    if (r.event_type !== 'chat.tool_result' || !r.tool_call_id) continue;
    const callRec = records.find(x => x.event_type === 'chat.tool_call' && x.tool_call?.id === r.tool_call_id);
    if (!callRec) continue;
    const agent = agentOf(callRec);
    // aggregated: consecutive same tool → cycle edge (handled by seq dedup); expanded: pair to result follower
    const idx = records.findIndex(x => x.id === r.id);
    for (let j = idx + 1; j < records.length; j++) {
      const nx = records[j];
      if (nx.event_type === 'chat.tool_call' || nx.event_type === 'chat.usage_metadata' ||
          (nx.event_type === 'chat.final' && (nx.content ?? '').trim())) {
        const fromKey = mode === 'aggregated'
          ? `tool:${agent ?? ''}:${callRec.tool_call?.name || callRec.tool_name || 'tool'}`
          : undefined; // expanded keys are positional; pair edge optional detail
        if (fromKey && nodes.get(fromKey)) edges.push({ from: fromKey, to: fromKey, kind: 'cycle' });
        break;
      }
    }
  }

  return { nodes: [...nodes.values()], edges };
}
```

Note: the test asserts only the documented invariants (collapse + counter + ≥1 cycle in aggregated; one node per call + pair edge presence in expanded). Adjust internals freely as long as those hold; delete dead branches rather than leaving unreachable code (`noUnusedLocals`).

- [ ] **Step 4: Run to verify it passes** — `npm run test:trace-graph`. Expected: 2 passing.

- [ ] **Step 5: Implement `TraceGraph.tsx` renderer**

Layered left-to-right SVG: rank = position in topological order of `seq` edges (nodes not in edges get rank 0); member lanes = group by `agent`, ordered by first appearance; node = rounded rect (48×28) with label + `(count/…)` counter when `count > 1`; edge = cubic bezier, `cycle` edges drawn as a self-loop arc. Colors: node stroke `cat(index of agent lane)`, fill `C.surface`; `llm` nodes `C.violet`, `tool` `C.warn`, `final` `C.info`, `user` `C.text`. Mode toggle buttons (`Aggregated | Expanded`) persist to `localStorage['tracehound.graphMode']`. Node click → `onSelectRecord(recordIds[0])` prop.

Keep the whole component under ~150 lines; viewBox sized `nodes.length * 120` wide, `lanes * 64` tall, wrapped in a horizontally scrollable div (`overflowX: 'auto'`).

- [ ] **Step 6: Tabs in `TurnDetailView`**

```tsx
const [tab, setTab] = useState<'records' | 'graph'>('records');
// header row:
<div style={{ display: 'flex', gap: 6, marginBottom: 10 }}>
  {(['records', 'graph'] as const).map(k => (
    <button key={k} onClick={() => setTab(k)} style={{
      fontSize: 12, padding: '4px 12px', borderRadius: 8, cursor: 'pointer',
      border: `1px solid ${tab === k ? C.info : C.border}`,
      background: tab === k ? C.infoSubtle : C.surface,
      color: tab === k ? C.info : C.textMuted, fontWeight: tab === k ? 600 : 400,
    }}>{t(`traceHound.graph.${k}`)}</button>
  ))}
</div>
{tab === 'records' ? <ExistingRecordsStack /> : (
  <TraceGraph records={turnRecords} onSelectRecord={id => setTab('records') || scrollToRecord(id)} />
)}
```

- [ ] **Step 7: Verify + commit**

Run: `cd jiuwenswarm/channels/web/frontend && npx tsc --noEmit && npm run build && npm run test:trace-graph`

```bash
git add jiuwenswarm/channels/web/frontend/src/components/TraceHound/traceGraph.ts \
        jiuwenswarm/channels/web/frontend/src/components/TraceHound/TraceGraph.tsx \
        jiuwenswarm/channels/web/frontend/src/components/TraceHound/index.tsx \
        jiuwenswarm/channels/web/frontend/tests/traceGraph.test.mjs \
        jiuwenswarm/channels/web/frontend/package.json
git commit -m "feat(tracehound): Langfuse-style agent graph per turn (aggregated/expanded)"
```

---

### Task 11: Final verification sweep (no new code)

- [ ] **Step 1: Full builds**

```bash
cd jiuwenswarm && python -m pytest tests/unit_tests/server -v
cd jiuwenswarm/channels/web/frontend && npx tsc --noEmit && npm run build \
  && npm run test:trace-live && npm run test:trace-highlights && npm run test:trace-graph
```

Expected: pytest all pass; tsc exit 0; build success; 6 node tests pass.

- [ ] **Step 2: Playwright walkthrough (localhost:5173)**

1. Open an old team session (Prague) → trajectory header shows `—` for LLM/tokens; no polling traffic in the network tab (flag default OFF).
2. Set `tracehound.live_updates_enabled: true` in the user's config, restart backend, open a fresh team session → run a prompt → LIVE pulse appears, turn list grows without manual refresh.
3. Highlights strip present and clickable on the Prague session.
4. Timeline band renders in turn detail; clicking a dot scrolls to the record.
5. Stats tool row with failures jumps to the failing record.
6. Graph tab: Aggregated collapses repeats with counters/cycles; Expanded unrolls; toggle persists after reload.
7. Switch language en↔zh — every TraceHound string localized.
8. Per-agent card shows llm/tok for the fresh session; `—` for Prague.

- [ ] **Step 3: Final commit if anything needed touching up**

```bash
git status   # confirm only intended files ever staged; nothing from the junk list
```
