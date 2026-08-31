# TraceHound Trajectory — More Informative & Engaging (Design Spec)

- **Date:** 2026-08-27
- **Status:** Approved design (section-by-section review with user); awaiting implementation order
- **Scope:** `jiuwenswarm/channels/web/frontend` (TraceHound trajectory panel) + `jiuwenswarm/server` (usage aggregation, mtime action, config flag)

## 1. Context

TraceHound's trajectory is surfaced inside the chat window (right slot, opened from the
chat-header paw button — commit `8e1e7d38`). A live review of the Prague team session
(`web_1a03e4eba55_beccebff91af`) surfaced these problems:

1. **Wrong stats for team sessions.** Header shows `0 LLM calls · 0 tokens` despite
   45 attempts and 237 events.
2. **No live updates.** The panel is a static snapshot.
3. **Dense, all-text Stats.** 3px CSS bars, no timeline, no per-agent breakdown.
4. **Not localized.** Every string is hardcoded English; the app is en/zh.
5. **Hardcoded hex colors** throughout — violates frontend `AGENTS.md`
   (must use `--color-*` theme tokens).
6. **No free guidance.** The only "explain this" feature (Diagnosis) costs LLM tokens.

### Verified data facts (grounding for the design)

- The godview history (`~/.jiuwenswarm/agent/sessions/<sid>/history.jsonl`) for the
  Prague session contains **zero** `chat.usage_metadata` / `chat.usage_summary` events.
  A whole-disk search found usage events only in two old **single-agent** sessions.
  Team-member LLM usage is not durably persisted anywhere.
- Member events mirrored into the godview history **carry the parent turn's
  `request_id` + `member_name`** (e.g. `prague-foodie` tool_call → `req_mta588l2_32`,
  turn #1). Stray `team.task-*` request_ids are single ghost records and are already
  filtered by `_is_real_turn`.
- `read_team_history_records` and `read_session_history_records` read the **same file**
  (different filters). There are no member sub-session usage files to read.
- `chat.usage_metadata` / `chat.usage_summary` are emitted in
  `server/runtime/agent_adapter/interface_deep.py` (~lines 11571 / 11941).
- `_replay_build_turns` (`server/agent_ws_server.py:10634`) keys turns by `request_id`
  and already aggregates usage events **when they exist**; its usage branches never call
  `_record_agent` (no per-agent credit).
- The `stats_only` param of `tracehound.turns.list` still runs the full
  read + replay server-side before trimming the response — it is **not** a cheap poll.
- `get_history_mtime(session_id)` exists (`server/runtime/session/session_history.py:284`).
- Runtime config flags ride the `config.get` whitelist (built in
  `gateway/channel_manager/tui/tui_connect.py:862`, `_config_get`; values `"true"/"false"`),
  are fetched once by `App.tsx` into `serverConfig`.
- Frontend: light-only theme (`src/styles/themes/default/light.css`, 191 tokens);
  no chart library; `noUnusedLocals/Parameters` are on; pytest 9.0.3 available
  (no conftest — run `python -m pytest`); `_replay_build_turns` is a bound method on
  `AgentWSServer` (test via `AgentWSServer.__new__` or a pure helper).

## 2. Goals / Non-goals

**Goals**
1. Accurate LLM/tokens/cost/latency/model stats for team sessions (forward-looking).
2. Observability-light palette via role-named theme tokens (Datadog/Grafana feel,
   not fancy).
3. Free (no-LLM) "highlights" strip of top signals.
4. SVG timeline of turn events + per-agent activity charts.
5. Live updates for the active session — **behind a config flag, default OFF**.
6. Cross-links from Stats (failing tools, outcome bars) into turn detail.
7. Full en/zh i18n and hex→token migration; remove dead nav keys.
8. Langfuse-style **agent graph** per user message (`Records | Graph` tabs,
   Aggregated/Expanded views).

**Non-goals**
- Dark theme (app is light-only).
- Retroactive backfill of historical team sessions (data was never written).
- Adding a chart/graph library (hand-rolled SVG only).
- Splitting the 2300-line `TraceHound/index.tsx` into multiple files.
- Session-level agent graph (per-turn only; session-level is a possible follow-up).

## 3. Phase 1 — Foundation

### 3.1 Trace theme tokens

**Files:** `src/styles/themes/default/light.css`, new `src/components/TraceHound/traceTokens.ts`.

Role-named tokens (AGENTS.md: names by role, not hue), with `-subtle` variants:

| Token | Role |
|---|---|
| `--color-trace-ok` / `-subtle` | completed, success |
| `--color-trace-warn` / `-subtle` | problems, retries, slow |
| `--color-trace-danger` / `-subtle` | errors, tool failures |
| `--color-trace-violet` / `-subtle` | LLM calls, skills, deferred |
| `--color-trace-info` / `-subtle` | files, general info |
| `--color-trace-teal` | artifacts |
| `--color-trace-cat-1…6` | categorical scale (agents, query types) |
| `--color-trace-surface`, `-surface-muted`, `-border`, `-border-strong`, `-text`, `-text-muted`, `-text-faint` | neutrals |

`traceTokens.ts` exports `const C = { ok: 'var(--color-trace-ok)', … }` because the
component uses inline styles (inline styles may reference CSS vars). All TraceHound
components migrate hex → `C.*`; panel background moves to the observability
`surface` family.

### 3.2 Backend usage fix (write-side, forward-looking)

**Root cause:** teammate LLM usage events are never persisted; the godview history
only receives mirrored tool/final events.

**Fix:**
1. **Write side** — reuse the existing member-event forwarding path (the mechanism
   that mirrors teammate `chat.tool_call`/`chat.tool_result` into the parent godview
   history and stamps parent `request_id` + `member_name`) to also forward
   `chat.usage_metadata` / `chat.usage_summary`. Implementation spike: locate the
   exact forwarding function (team mirror path; candidate area
   `server/runtime/agent_adapter/team_helpers.py` + `interface_deep.py` emission
   points) and extend its event-type allowlist.
2. **Replay side** (`_replay_build_turns`) — because forwarded events carry the parent
   `request_id`, the existing `usage_metadata`/`usage_summary` branches attribute usage
   to the correct turn automatically. **No time-window attribution needed.** Only
   change: those branches call `_record_agent(...)` to credit per-agent
   `llm_calls` / `tokens` / `cost`; extend the `AgentActivity` payload accordingly
   (backend dict + frontend `AgentActivity` interface + per-agent chart consumption).
3. `session_stats`, the metadata cache, and `stats_only` inherit corrected sums
   automatically (they aggregate from `real_turns`).
4. **Historical sessions are not backfilled.** The UI must show `—` (and hide
   LLM/token/cost chips) when a session has no usage records at all — never a
   misleading `0`.

**Test:** new pytest `tests/unit_tests/server/test_tracehound_replay_aggregation.py`:
build synthetic parent-history records (user msg, member tool_call/tool_result with
`member_name` + parent request_id, forwarded usage events with `member_name`) and
assert turn-level `llm_call_count`/`total_tokens` and per-agent credits. Instantiate
via `AgentWebSocketServer.__new__(AgentWebSocketServer)` (skip `__init__`; the class lives in
`server/agent_ws_server.py`, all replay helpers are class attrs/args) or extract a pure helper
if cleaner.

## 4. Phase 2 — Features (built on correct data)

### 4.1 Highlights strip (free, no LLM)

Horizontal strip under the panel header (above Diagnosis). Deterministic top 3–4
signals computed from `turns`: most-retried message, top failing tools, slowest
message, context pressure, "N of M with problems". Each chip is clickable — filters
the user-message list or jumps to turn detail. Hidden entirely when nothing notable.

### 4.2 Timeline + per-agent charts

- **Timeline:** hand-rolled SVG strip per turn (x = wall-clock time within the turn),
  colored dots/segments per event type (tool call/result, LLM call, final, error),
  tooltips, click → jump to that record in `Records` view. Time-bucketing for dense
  turns; clean zero states.
- **Per-agent activity:** new Stats card (team sessions only) — grouped bars per
  member: tool calls, failures, LLM calls, responses, tokens (uses 3.2 data),
  categorical `--color-trace-cat-*` colors, member lanes ordered by spawn sequence.
- Shared SVG helper module (used again by Phase 4).

### 4.3 Live updates (feature-flagged, default OFF)

- **config.yaml:** new top-level `tracehound:` section with
  `live_updates_enabled: false` (ship in `resources/config.yaml`; missing key ⇒ false).
- **Backend:**
  - `_config_get` whitelist gains `tracehound_live_updates_enabled`
    (`"true"/"false"` string per existing convention).
  - New request `tracehound.session.mtime` → returns `get_history_mtime(session_id)`:
    add `ReqMethod` in `common/schema/message.py`, an action branch in
    `_handle_replay_request` (`server/agent_ws_server.py`), and register in the web
    handler allowlists (`gateway/channel_manager/web/app_web_handlers.py`, both lists).
- **Frontend:** `App.tsx` passes `liveUpdatesEnabled` (from `serverConfig`) to
  `TrajectoryPanel`. When ON and the panel shows the active session and is connected:
  poll **mtime** every ~5s (cheap); only when it changes, call `tracehound.turns.list`
  via a new `refreshTurns()` store action that updates `turns`/`sessionStats`
  **in place** (preserves `selectedTurnId`, filters, scroll). If a turn detail is open
  and its `event_count` grew, re-fetch its records too. Pause polling while
  `analyzing` is true. Show a "● LIVE" pulse in the panel header only while enabled.
  Rationale: `stats_only` is not cheap (full replay server-side), so mtime gates it.

### 4.4 Cross-links (Stats → detail)

- Clicking a tool row with failures in the Stats tool-usage card opens the first turn
  containing that failure (`selectTurn`) pre-scrolled to the failing
  `chat.tool_result` record (store gains a `focusRecordId` consumed by
  `TurnDetailView`).
- Outcome bars / highlight chips jump to the corresponding turn.

## 5. Phase 3 — i18n + polish

- Move all TraceHound strings to `traceHound.*` keys in `src/i18n/locales/{en,zh}.json`
  (labels, filters, tooltips, empty states, header, detail view, graph UI).
- Complete the hex→`C.*` token migration (started in 3.1).
- List polish: cards/badges/filters on the new palette; collapse/filter/embedded
  behavior unchanged.
- Cleanup: remove dead `nav.tracehound` keys from both locales; optionally drop
  `'tracehound'` from `MainNavKey` (`src/utils/frontendPlatform.ts`).

## 6. Phase 4 — Agent graph (Langfuse-style, per turn)

**New** `src/components/TraceHound/TraceGraph.tsx`; `TurnDetailView` gains
`Records | Graph` tabs (Records behavior unchanged).

- **Graph model:** nodes = user input, member/leader agents, tool calls, LLM calls,
  final output; edges inferred from (a) record timestamps within the turn,
  (b) `tool_call_id` pairing of call→result, (c) LLM call → following tool call,
  (d) leader→member spawn links (`spawn_teammate` + `member_name` lanes).
- **Aggregated view:** one node per (agent, step name) with run counters
  (`retrieve_docs (3/3)`), repeated steps drawn as cycle edges.
- **Expanded view:** one node per call, execution order DAG.
- Toggle persisted in `localStorage` (like Langfuse remembers the choice).
- **Layout:** layered SVG (rank by depth), member lanes ordered by spawn sequence,
  curved edges, hover tooltips (step/agent/usage — benefits from 3.2 data),
  click → switch to `Records` tab at that record. Pan/zoom for dense turns.
- Reuses Phase 2 SVG helpers. Honest caveat: history is a linear replay, so edges are
  inferred (temporal + pairing), not true nesting; lane layout mitigates interleaving.

## 7. Verification

- Frontend: `cd jiuwenswarm/channels/web/frontend && npx tsc --noEmit && npm run build`.
- Backend: `python -m pytest tests/unit_tests/server/test_tracehound_replay_aggregation.py`.
- Playwright walkthrough on localhost:
  - fresh team session (post-fix): LLM calls / tokens / cost > 0, per-agent chart
    populated, per-agent tooltips show usage credits;
  - old Prague session: graceful `—`, no polling;
  - flag OFF (default): no mtime polling traffic; flag ON: LIVE pulse + auto-refresh
    while the session progresses;
  - highlights, timeline, cross-links, graph both views (Aggregated/Expanded),
    tab persistence, click-through to records;
  - en + zh locales render fully.

## 8. Risks & mitigations

| Risk | Mitigation |
|---|---|
| Write-side forwarding touches the deep-agent hot path | Guarded try/except; event-type allowlist extension only; verified by fresh team run |
| Replay/unit test needs heavy `AgentWSServer` | `__new__` instantiation or pure helper extraction |
| Polling load | mtime-gated (no full replay per tick), paused while analyzing, flag default OFF |
| Graph layout quality on dense turns | Lane-based layered layout, node caps, pan/zoom, aggregated default |
| i18n surface size | Mechanical key-per-string pass; no behavior change |

## 9. Explicitly out of scope / future work

- Session-level agent graph.
- Dark-theme trace tokens.
- Retroactive team-usage backfill (data was never persisted).
