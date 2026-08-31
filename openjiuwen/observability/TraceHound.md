# TraceHound

TraceHound is JiuwenSwarm's built-in session trajectory viewer and analyser. It lets you inspect any past agent session turn-by-turn, measure performance, diagnose failures, and run an LLM-powered deep analysis to surface improvement opportunities — all without re-running the agent.

---

## Concepts

### What is a Turn

A **turn** is a single request/response cycle inside a session. It starts when the user sends a message and ends when the agent produces its final response (`chat.final`). One turn can contain many internal events: LLM calls, tool invocations, file reads, error recoveries, and streaming deltas.

TraceHound groups all history records that share the same `request_id` into a single turn and computes aggregated metrics for it.

### What TraceHound Records

Every event the agent emits is appended to a history file on disk in real time. TraceHound reads this file — it does not require any additional instrumentation. The event types recorded include:

| Event type | Description |
|------------|-------------|
| `chat.message` | Incoming user message |
| `chat.tool_call` | Tool the agent chose to invoke |
| `chat.tool_result` | Result returned by the tool |
| `chat.final` | Agent's completed response text |
| `chat.error` | Error raised during the turn |
| `chat.usage_metadata` | LLM call metrics: tokens, latency, cost |
| `chat.usage_summary` | Per-turn token / cost totals |
| `chat.reasoning` | Internal reasoning trace (when extended thinking is active) |
| `chat.delta` | Streaming output chunk |
| `chat.file` | File attachment metadata |

### Turn Outcome Classification

TraceHound classifies every turn into one of five outcomes:

| Outcome | Meaning |
|---------|---------|
| `completed` | Agent produced a final response with no errors |
| `completed_with_issues` | Final response present but retries, tool failures, or warnings occurred |
| `no_response` | No `chat.final` event — agent did not finish the turn |
| `error` | One or more `chat.error` events; agent may or may not have recovered |
| `deferred` | Message was queued but never processed |

### Error Categories

When an error is detected, TraceHound categorises it automatically:

| Category | Triggered by |
|----------|-------------|
| `api_auth` | Authentication or authorisation failures |
| `timeout` | Timeout or deadline exceeded errors |
| `filesystem` | File not found, permission denied, I/O errors |
| `network` | Connection refused, DNS, SSL errors |
| `syntax` | SyntaxError or parse failures |
| `import` | Module import failures |
| `model` | Model-specific or context-length errors |
| `execution` | Runtime exceptions not covered above |
| `other` | Uncategorised errors |

### Query Type Classification

Each user message is also classified by intent:

| Type | Description |
|------|-------------|
| `debug` | Debugging or troubleshooting request |
| `file_op` | File read, write, create, or delete |
| `coding` | Code generation or modification |
| `analysis` | Data analysis or evaluation |
| `question` | Factual question or explanation |
| `general` | Everything else |

---

## Accessing TraceHound

TraceHound is accessible from the **TraceHound** tab in the left sidebar of the JiuwenSwarm web UI. No configuration or installation is needed — it is built into the core web server and is always available.

---

## Three-Level Navigation

TraceHound uses a three-level drill-down:

```
Session List  →  Turn List  →  Turn Detail
```

### Session List

The entry view shows all recorded sessions. Each row displays:

- Session title (or raw session ID if no title is set)
- Agent mode badge (`agent.plan`, `agent.fast`, `team`)
- Time of last message
- Summary statistics: user messages, LLM calls, events, tokens, cache tokens, cost, average latency, context usage %, and models used

Use the **"Show empty sessions"** toggle to include or exclude sessions that have no turns.

Click a session to advance to the Turn List.

### Turn List

The Turn List is the main analytics view for a selected session. It has three sections:

#### 1. Session Stats Bar

A row of metric chips at the top of the view:

| Chip | Description |
|------|-------------|
| Total turns | Number of turns in the session |
| Errors | Turns that had at least one error event |
| Total tokens | Sum of all tokens across all turns |
| LLM calls | Total number of LLM API calls |
| Total events | Total event count in the history file |
| Cache tokens | Tokens served from the prompt cache |
| Input / Output cost | Dollar cost breakdown |
| Avg latency | Average total LLM call latency across turns |
| Avg TTFT | Average time-to-first-token |
| Avg TPOT | Average time-per-output-token |
| Max context | Highest context window usage % across turns |
| Date range | First to last turn timestamp |

#### 2. Analytics Charts

A set of visual breakdowns:

| Chart | Description |
|-------|-------------|
| Outcome distribution | Count of turns by outcome category |
| Token usage per turn | Bar chart of total tokens per turn |
| LLM calls per turn | Bar chart of LLM call count per turn |
| Retry loop count | Turns that triggered retry logic |
| Turn duration | Wall-clock time per turn |
| Error categories | Pie chart of error type distribution |
| Query type distribution | Breakdown of user intent classifications |
| Tool usage (top 8) | Most frequently called tools |
| Skill usage (top 8) | Most frequently used skills |

#### 3. Turn Cards

Each turn is shown as a card with:

- User message excerpt and timestamp
- Outcome badge and issue count indicator
- Token count, LLM call count, duration, retry count, tool failure count
- Tool names called in the turn

**Filter bar**: Toggle visibility by outcome type, presence of retry loops, or slow turns (turns above the p90 duration threshold).

Click a turn card to advance to Turn Detail.

#### 4. LLM Analysis Panel

See [LLM-Powered Analysis](#llm-powered-analysis) below.

### Turn Detail

The deepest level — a chronological timeline of every event recorded in the turn.

**Turn header** shows: turn index, user message, outcome badge, wall-clock duration, and total token count.

**Export button** (`⬇ JSON`): downloads the full turn record as a JSON file.

**Per-event cards** are rendered by type:

| Event type | What is shown |
|------------|---------------|
| `chat.final` | Agent response text; response-length badge (`terse` / `normal` / `verbose` / `essay`) |
| `chat.tool_call` | Tool name, arguments; dangerous command warning when applicable |
| `chat.tool_result` | Tool result; error status; retry badge |
| `chat.usage_metadata` | Model name, token counts, TTFT, TPOT, total latency, cost |
| `chat.error` | Error message and error category badge |
| `chat.reasoning` | Collapsible internal reasoning trace |
| `chat.delta` | Streaming output chunk (for debugging streaming issues) |

Every card also shows two timing annotations:
- **Δ prev** — seconds since the previous event in the turn
- **Elapsed** — seconds since the first event in the turn

Use **← Back** to return to the Turn List.

---

## LLM-Powered Analysis

The **LLM Analysis** panel in the Turn List view sends a compact summary of the session to a language model and asks it to identify issues and suggest improvements.

### How to Run

Expand the **LLM Analysis** section in the Turn List view and click **Analyse**. The request is asynchronous — results appear within a few seconds.

### What the Analysis Returns

The model returns a ranked list of issues, each with:

| Field | Description |
|-------|-------------|
| **Priority** | 1 (critical) to 5 (informational) |
| **Title** | Short summary of the issue (up to 70 characters) |
| **Description** | Clear explanation of what was observed |
| **Evidence** | Specific facts: timestamps, error messages, counts |
| **Impact** | Effect on the user or the system |
| **Root cause** | Likely underlying reason |
| **Recommendation** | Actionable steps to fix or mitigate the issue |

Issues are colour-coded by priority. Click any issue card to expand its full detail.

### Caching

Analysis results are cached in the browser's `localStorage` with a staleness key derived from the history file's size and modification time. If the session history changes (new turns appended), the cache is automatically invalidated on the next visit and a fresh analysis is offered.

> **Note:** LLM analysis requires a model to be configured in your JiuwenSwarm settings. If no model is available the request returns an error and the panel displays a configuration prompt.

---

## Storage & Data Format

### History File

All agent events are written to disk as they occur:

- **Default format** — JSONL (`history.jsonl`): one JSON object per line, append-only.
- **Legacy format** — JSON array (`history.json`): enabled by setting the environment variable `JIUWENSWARM_USE_LEGACY_HISTORY_JSON=1`.

**File location:**

```
~/.jiuwenswarm/agent/sessions/<session_id>/history.jsonl
```

**Example records (JSONL):**

```jsonl
{"role":"user","event_type":"chat.message","request_id":"req-1","content":"Refactor the auth module","timestamp":1700000000.0}
{"role":"assistant","event_type":"chat.tool_call","request_id":"req-1","tool_name":"str_replace_editor","tool_call":{"id":"tc-1","name":"str_replace_editor","arguments":{"command":"view","path":"/src/auth.py"}},"timestamp":1700000001.2}
{"role":"user","event_type":"chat.tool_result","request_id":"req-1","tool_name":"str_replace_editor","content":"...file contents...","timestamp":1700000001.8}
{"role":"assistant","event_type":"chat.usage_metadata","request_id":"req-1","total_tokens":4200,"ttft_ms":320,"tpot_ms":18,"total_latency_ms":1100,"timestamp":1700000003.5}
{"role":"assistant","event_type":"chat.final","request_id":"req-1","content":"I've refactored the auth module. Here is what changed…","timestamp":1700000003.6}
```

### Session Metadata Cache

Each session directory also contains a `metadata.json` file. TraceHound updates several fields in this file after a `turns_list` request so that the Session List can display statistics without re-scanning the full history:

```
~/.jiuwenswarm/agent/sessions/<session_id>/metadata.json
```

Fields written by TraceHound:

| Field | Description |
|-------|-------------|
| `total_tokens` | Sum of tokens across all turns |
| `round_id` | Total number of turns |
| `llm_calls` | Total LLM API calls |
| `total_events` | Total event count |
| `total_cache_tokens` | Cache-served tokens |
| `total_input_cost` / `total_output_cost` / `total_cost` | Dollar costs |
| `models_used` | List of distinct model names used |
| `avg_total_latency_ms` | Mean LLM call latency |
| `avg_ttft_ms` | Mean time-to-first-token |
| `avg_tpot_ms` | Mean time-per-output-token |
| `max_context_usage_percent` | Highest context window utilisation |
| `channel_id` | Channel that originated the session |

---

## WebSocket API

TraceHound is accessed via the same WebSocket connection as the rest of the JiuwenSwarm server. Three request methods are exposed:

### `tracehound.turns.list`

Returns the list of turn summaries and session-level aggregate statistics for a given session.

**Request params:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `session_id` | string | yes | Target session ID |
| `stats_only` | boolean | no | If `true`, return only `session_stats` without the full `turns` array |

**Response:**

```json
{
  "ok": true,
  "turns": [ /* array of TurnSummary objects */ ],
  "session_stats": { /* SessionStats object */ }
}
```

### `tracehound.turn.get`

Returns the full ordered list of history records that belong to a single turn.

**Request params:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `session_id` | string | yes | Target session ID |
| `turn_id` | string | yes | `request_id` of the turn to fetch |

**Response:**

```json
{
  "ok": true,
  "records": [ /* array of HistoryRecord objects ordered by timestamp */ ]
}
```

### `tracehound.analyze`

Runs the LLM-powered analysis against a session's history.

**Request params:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `session_id` | string | yes | Target session ID |

**Response:**

```json
{
  "ok": true,
  "issues": [
    {
      "priority": 1,
      "title": "Repeated tool call failures on str_replace_editor",
      "description": "...",
      "evidence": "...",
      "impact": "...",
      "root_cause": "...",
      "recommendation": "..."
    }
  ],
  "fingerprint": "83421:1700012345"
}
```

The `fingerprint` value is `<file_size_bytes>:<mtime_unix>`. Clients should cache the response keyed on `session_id` and invalidate when the fingerprint changes.

---

## FAQ

### Q1: Do I need to configure anything to use TraceHound?

No. TraceHound is built into the JiuwenSwarm web server and is always available. The only optional dependency is an LLM model for the analysis feature — without one, the LLM Analysis panel will show an error message when you try to run an analysis.

### Q2: Does TraceHound slow down the agent?

No. TraceHound reads the history files that the agent writes anyway. There is no additional instrumentation and no in-path overhead. The `tracehound.turns.list` scan runs on-demand when you open a session in the UI.

### Q3: Why does a turn show outcome `no_response`?

The `no_response` outcome means the turn has no `chat.final` event. This happens when:
- The server was stopped or crashed before the agent completed the turn.
- The agent was interrupted by the user.
- A fatal error prevented the agent from reaching the final step.

Check the **Turn Detail** view for `chat.error` events that may explain the interruption.

### Q4: How do I export a turn for bug reporting?

Open the **Turn Detail** view for the turn and click **⬇ JSON**. This downloads the complete record set for that turn as a JSON file, which you can attach to an issue report.

### Q5: The "Loading history…" indicator appeared in the chat panel — is that related to TraceHound?

No. That indicator is shown by the IDE plugins (JetBrains and VS Code) when they fetch session history after you switch sessions. It uses the same underlying history infrastructure but is separate from the TraceHound UI.

### Q6: Where is the LLM analysis cache stored?

In the browser's `localStorage` under the key `tracehound_analysis_<session_id>`. It is automatically cleared when the history file changes. To force a fresh analysis, clear your browser's local storage or wait for the history to be updated by a new turn.

### Q7: Can I use TraceHound on very large sessions?

Yes, but note that the `tracehound.turns.list` call performs a linear scan of the full history file. For sessions with thousands of events this may take a second or two on first load. Subsequent loads are faster because session statistics are cached in `metadata.json`.

---

## Related Links

- [Session management](Session.md) — Understanding session storage and lifecycle
- [Logging System](Logs.md) — Agent server and gateway logs
- [Page Overview](Page-Overview.md) — Web UI layout and navigation
- [E2A Protocol](E2A-protocol.md) — WebSocket protocol used by TraceHound's API methods
- [Configuration](Configuration.md) — Configuring the LLM model used for analysis

---

*Document Version: v1.0*
*Target Audience: JiuwenSwarm Users and Developers*
*Last Updated: 2026-07-12*
