# Agent-session provider — design (pass 1 of 4)

Brief: `docs/agent-session/00-brief.md` (authoritative). Base: upstream jacob-bd/the-ai-counsel @ 614dfb9, branch
`feat/agent-session-provider`. Every statement about current behaviour cites `file:line` at that commit. Inferences
are marked "(inferred)". Nothing here is code; JSON shapes and config snippets only.

## 1. Summary

A new provider, key `agent-session`, model ids `agent-session:<slot-id>`, is added to the existing provider registry
so the council, advisors, debate and preflight code call it exactly as they call any other provider. Each model id is
a *slot*. A live Claude Code, Codex CLI or local-model runner process occupies a slot by calling MCP tools on The AI
Counsel's own MCP server: take, wait (long-poll), answer (optionally in chunks), heartbeat, release, log. The
provider's `query()` posts the new question to the slot's in-memory queue, wakes the session's pending wait, and
blocks until the final answer chunk or the per-provider timeout. An empty slot fails immediately with a distinct
error. The MCP server gains a streamable-HTTP endpoint at `POST /mcp` on the existing backend port; the tools reach
the slot registry through new REST routes on the same FastAPI process, as every existing tool already does. A
per-slot JSONL log on disk records every exchange. A "send to session" action posts a verdict as a slot's next task.

## 2. Goals and non-goals

Goals (brief line in brackets):
- G1 The council's ask-and-wait loop is untouched; the change is one more `LLMProvider` [9, 16].
- G2 A Claude Code or Codex session is a model; it is never pushed to, only served when it is waiting [10, 11-12].
- G3 One slot per session; MCP tools list/take/wait/answer/heartbeat; streamable HTTP on the existing address [17-19].
- G4 Occupied = recent heartbeat or currently waiting; empty slot fails at once; take returns the slot id [20-21].
- G5 Preflight "Reply with OK." is delivered to the session; no auto-reply anywhere [22].
- G6 Session receives only the new turn (+ system prompt); full exchange logged per slot and readable [23].
- G7 Answer chunks accepted as they arrive; the caller waits for the whole answer; timeout long enough [24-25].
- G8 No password, no TLS on this path [26].
- G9 A skill drives the session loop without hooks [27].
- G10 Subscriptions only (Claude via a session, Copilot/ChatGPT via existing OAuth) [30-31]; local model runner [32];
  verdict sent to a chosen slot as its next task [33-34].

Non-goals:
- N1 Reusing any code from SmoothSDLC/agent-chatroom [13].
- N2 Waking a session that is not in a wait call [11]; no push channel, no webhooks.
- N3 Authentication, TLS, multi-worker deployments (the backend already keeps run state process-local, `backend/main.py:73-76`).
- N4 A streaming UI for partial answers; only the plumbing that a future streaming caller would use (section 5.4).
- N5 Changing preflight semantics for other providers.

## 3. Glossary

- **provider**: a Python class implementing `LLMProvider` (`backend/providers/base.py:6-45`), registered under a key in
  `PROVIDERS` (`backend/council.py:36-52`).
- **model id**: string `"<provider-key>:<model>"`; the part before the first `:` selects the provider, anything else
  falls to `openrouter` (`backend/council.py:54-60`). Here: `agent-session:<slot-id>`.
- **slot**: a named, configured seat that exactly one session occupies at a time. Its model id is `agent-session:<slot-id>`.
- **session**: a running Claude Code / Codex CLI / local runner process connected to the MCP server and looping on the skill.
- **lease**: an opaque id issued on take that identifies the current occupant of a slot (not a secret; see 5.2).
- **question**: one unit of work posted to a slot by `query()` or by "send to session"; has a `question_id`, a kind
  (`council`, `preflight`, `task`), a deadline.
- **answer**: text returned by the session for a question; may arrive as several chunks, the last marked `final`.
- **wait**: the long-poll MCP call a session sits in; returns the instant a question is posted, or `idle` after N seconds.
- **heartbeat**: a call that refreshes the slot's `last_seen` time; wait, answer and heartbeat all count.
- **occupied**: slot has `last_seen` within the heartbeat window, or a session in wait, or an in-flight question before its deadline.
- **preflight**: the council's availability ping, one user message `"Reply with OK."`, 5 s timeout
  (`backend/model_preflight.py:15-16,110-118`).
- **streaming**: answer chunks appended as they arrive; today's callers only consume the final concatenation.

## 4. Current state with evidence

**Provider registration and selection.** `backend/providers/__init__.py` is empty (0 lines). Providers are
instantiated once at import into the module-level dict `PROVIDERS` in `backend/council.py:36-52` (15 keys, e.g.
`"ollama": OllamaProvider()`, `"github-copilot": GitHubCopilotProvider()`). `get_provider_name_for_model` returns the
prefix before `:` if it is a key of `PROVIDERS`, else `"openrouter"` (`council.py:54-60`); `get_provider_for_model`
indexes the dict (`council.py:63-65`). `backend/main.py:2176-2194` (`GET /api/models/direct`) iterates every
`PROVIDERS` entry except `openrouter`, `ollama`, `hybrid` and concatenates each `get_models()` result, ignoring
enabled toggles. `POST /api/settings/test-provider` (`main.py:2354-2367`) rejects unknown keys, then returns
`"No API key provided or configured"` before calling `validate_key` when the resolved key is empty (`main.py:2363-2364`).

**`LLMProvider.query` contract.** Signature `query(model_id, messages, timeout=120.0, temperature=0.7) -> dict`
(`base.py:10`); docstring: dict with `content` or `error` + `error_message` (`base.py:20`). Concrete success shape:
`{"content": str, "usage": dict|None, "error": False}` (`backend/providers/custom_openai.py:66`); failure shape
`{"error": True, "error_message": str}` (`custom_openai.py:28,69`). OpenRouter uses `"error": None` on success and
`"error": "<code>"` on failure (`backend/openrouter.py:94-100,125-129`). The council only tests truthiness of `error`
and reads `error_message`, `usage`, `cost` (`council.py:263-272`), and stringifies `content` (`council.py:277-281`).
`get_models()` returns model dicts `{id, name, provider, is_free?}` (`base.py:25-32`; Ollama example
`backend/providers/ollama.py:39-44`). `validate_key(api_key)` returns `{success, message}` (`base.py:35-45`).

**Dispatch and timeouts.** `council.query_model` (`council.py:68-95`) resolves the provider, and when `timeout` is
`None` calls `request_timeout(provider_name)` (`council.py:82-83`); it special-cases `OpenCodeProvider` for a
`session_id` kwarg (`council.py:84-93`) and finally `attach_cost` (`council.py:94-95`). `request_timeout` precedence is
`{PROVIDER}_REQUEST_TIMEOUT` env → `LLM_COUNCIL_REQUEST_TIMEOUT` → `DEFAULT_REQUEST_TIMEOUT = 180.0`
(`backend/providers/timeouts.py:31,69-82`); the env name is the key upper-cased with `-`→`_`
(`timeouts.py:42-45`), so ours is `AGENT_SESSION_REQUEST_TIMEOUT`. Bounds 1–3600 s (`timeouts.py:36-37`). Every stage
wraps calls in `_query_safe` returning `{"error": True, "error_message": str(e)}` on exception (`council.py:232-241`)
and cancels pending tasks when the HTTP client disconnects (`council.py:250-254`). `attach_cost` normalises `usage`
(`{}` when `None`) and estimates cost (`backend/costs.py:713-721`); the provider prefix must be in
`_SUPPORTED_PROVIDER_PREFIXES` (`costs.py:32-48`) or it is treated as OpenRouter (`costs.py:138-144`); subscription
OAuth prefixes are priced free (`costs.py:49`; `skills/the-ai-counsel-api/SKILL.md:234`).

**Preflight.** `preflight_models` pings each selected model with `[{"role":"user","content":"Reply with OK."}]`,
`temperature=0.0`, default `timeout=5.0`, five at a time (`model_preflight.py:15-16,110-118,227`). A result whose
`error_message` contains "timeout"/"timed out" is a *soft* timeout — logged, run proceeds (`model_preflight.py:95-97,
135-137, 235-241`). Any other error is a hard failure that aborts the run with HTTP 400 or an SSE error event
(`model_preflight.py:244-245`; `main.py:617-629, 820-826`). The chairman is included in full mode (`main.py:572-577`).

**Model listing for the picker.** Backend: OpenRouter via `/api/models` (`main.py:2501`), Ollama `/api/ollama/tags`
(`main.py:2404`), custom `/api/custom-endpoint/models` (`main.py:2486`), everything else `/api/models/direct`
(`main.py:2176`). Settings payload exposes `enabled_providers` and `direct_provider_toggles`
(`backend/settings_payload.py:82-83`), defaults in `backend/settings.py:47-56,59-69`. Frontend: `CouncilSetup.jsx`
loads the four lists (`frontend/src/components/CouncilSetup.jsx:157-180`) and passes direct models through
`filterDirectModels` (`CouncilSetup.jsx:26-42`), which drops any model whose lower-cased `provider` is not a key in
`DIRECT_PROVIDER_KEY_FLAGS` (`CouncilSetup.jsx:11-24,39-41`) unless it carries an OAuth prefix handled by
`filterOAuthModels` (`frontend/src/constants/oauthProviders.js:23,31-41`). `AdvisorSetup.jsx:108-140,241-250` and
`Settings.jsx:1650-1680` repeat the pattern. **So a new provider's models are hidden until the frontend knows its
prefix** — the brief's "like any other model" needs the frontend touched (section 5.6). Group labels come from
`SearchableModelSelect.jsx:19-41,53-59`; grid icons from `frontend/src/utils/councilGridUtils.js:13-29,31-47,49-68`.

**How the MCP server runs.** `create_server()` builds one `FastMCP` and registers six tool modules
(`the_ai_counsel_mcp/server.py:15-53`). `backend/main.py:2595-2602` imports it and mounts `sse_app()` at `/mcp`
inside the FastAPI app, with `base_url=http://127.0.0.1:{BACKEND_PORT}`: **the MCP server runs in the FastAPI
process** on the backend port (`/mcp/sse`, `/mcp/messages`). The standalone entry point (`python -m
the_ai_counsel_mcp`) offers only `stdio` and `sse` (`the_ai_counsel_mcp/__main__.py:37-43`). Every tool talks to the
backend over HTTP through `CouncilClient` (`the_ai_counsel_mcp/client.py:19-33`, 180 s default timeout at `:22`),
never in-process, so tools work identically in mounted and stdio modes. No streamable-HTTP endpoint exists today, and
`app = FastAPI(title=...)` has no lifespan (`main.py:272`; no `lifespan`/`on_event` anywhere in `main.py`). The
installed SDK is `mcp 1.27.1` (`uv.lock:945-946`); `FastMCP.streamable_http_app()` exists and returns a Starlette app
whose lifespan runs `session_manager.run()` (`.venv/.../mcp/server/fastmcp/server.py:950,1044`), which may run only
once per instance (`.../mcp/server/streamable_http_manager.py:99-105`). Starlette does not run a mounted sub-app's
lifespan (inferred from Starlette's Mount semantics), so the host app must start it.

**Tool registration.** Each `tools/*.py` exposes `register(server, base_url)` and defines one action-based tool with a
`description` and an `action: str` argument returning a string, JSON when structured
(`the_ai_counsel_mcp/tools/providers.py:12-28`, `tools/conversations.py:10-21,80`). The repo convention is ten
consolidated tools (`docs/mcp/TOOLS.md:3`; `/api/health` reports `"tools": 10`, `main.py:707`).

**Streaming today.** `stream_buffer.py` converts backend SSE events into stage results for MCP callers
(`the_ai_counsel_mcp/stream_buffer.py:174,216,393`) and imports `backend.metadata_utils` (`:10`), so the MCP package
already depends on `backend`. Providers themselves do not stream; `query()` returns once.

**Storage.** Conversations are JSON files `data/conversations/{id}.json` (`backend/config.py:12`;
`backend/storage.py:62-64,387-403`, `json.dump(indent=2)`), plus an index file (`storage.py:12,67-69`). Settings live
in `data/settings.json` with an mtime cache (`settings.py:38,395-416`); `save_settings` redacts secret fields
(`settings.py:429-445`); pydantic defaults cover missing keys on load (`settings.py:406`). Docker mounts `./data`
(`docker-compose.yml:21`).

**Tests.** `uv run pytest` (`.github/workflows/docker-publish.yml:33`), `asyncio_mode = "auto"`
(`pyproject.toml:34`), dev deps pytest/pytest-asyncio/respx (`pyproject.toml:26-30`). Backend tests live in
`backend/tests/`; autouse fixtures redirect settings and credentials files to `tmp_path`
(`backend/tests/conftest.py:73-100,103-122`). Preflight tests patch `backend.model_preflight.query_model`
(`backend/tests/test_model_preflight.py:12-13`); route tests use `with TestClient(app, client=("127.0.0.1", 50000))`
(`backend/tests/test_settings_test_provider.py:24-27`); storage tests monkeypatch `storage.DATA_DIR`
(`backend/tests/test_storage_modes.py:7`). MCP tests build `create_server(base_url="http://test:8001")`, mock REST
with `respx`, and call `server.call_tool(...)` (`the_ai_counsel_mcp/tests/test_tools_council.py:10-12,18-40`), with
helpers `get_text`/`get_json` (`the_ai_counsel_mcp/tests/conftest.py:6-14`).

## 5. Target architecture

```
                 LAN, plain HTTP, one address: http://<host>:8001
 ┌──────────────────────────────── FastAPI process (backend.main:app) ─────────────────────────────────┐
 │  REST /api/*  ──────────────┐                                                                        │
 │  UI (frontend/dist)         │        ┌──────────────── provider registry PROVIDERS ───────────────┐  │
 │  /api/conversations/...     │        │ openai | anthropic | ... | ollama | github-copilot |        │  │
 │  council.py stage1/2/3 ─────┼──────▶ │ agent-session  (AgentSessionProvider.query)                │  │
 │  advisors.py / debate.py    │        └───────────────┬────────────────────────────────────────────┘  │
 │  model_preflight.py         │                        │ enqueue / await answer                        │
 │                             │        ┌───────────────▼──────────────┐    ┌────────────────────────┐  │
 │  /api/agent-sessions/* ─────┼──────▶ │ SlotRegistry (in memory)     │───▶│ data/agent_sessions/   │  │
 │   take/wait/answer/...      │        │ slots, leases, FIFO queues,  │    │ <slot-id>.jsonl (log)  │  │
 │                             │        │ asyncio events, partials     │    └────────────────────────┘  │
 │  /mcp  (streamable HTTP)    │        └──────────────────────────────┘                                │
 │  /mcp/sse (SSE, kept)  ◀────┼── FastMCP tools ── CouncilClient ──▶ http://127.0.0.1:8001/api/...     │
 └─────────────────────────────┼───────────────────────────────────────────────────────────────────────┘
                               │ MCP over streamable HTTP (POST /mcp), long-poll wait ≤ 90 s
        ┌──────────────────────┼─────────────────────┬──────────────────────────────┐
 ┌──────▼──────────┐  ┌────────▼─────────┐  ┌────────▼────────────┐   ┌─────────────▼──────────┐
 │ Claude Code     │  │ Codex CLI        │  │ local-model runner  │   │ browser UI             │
 │ skill loop:     │  │ skill loop       │  │ (any MCP client +   │   │ picker shows slots     │
 │ take→wait→      │  │ slot codex-1     │  │  Ollama/LM Studio)  │   │ occupied/empty; "Send  │
 │ answer→heartbeat│  │                  │  │ slot local-1        │   │ to session" on verdict │
 └─────────────────┘  └──────────────────┘  └─────────────────────┘   └────────────────────────┘
```

### 5.1 Slot registry

Location: new package `backend/agent_sessions/` (`registry.py`, `log.py`, `routes.py`) plus
`backend/providers/agent_session.py`; relative imports as required by `AGENTS.md` ("Python Module Imports").

Process model decision: **in-process, inside the FastAPI backend, reached by MCP tools through REST**. Evidence:
the MCP server is already mounted in the FastAPI process (`main.py:2595-2602`), and every existing tool already goes
through `CouncilClient` to `http://127.0.0.1:{BACKEND_PORT}` (`main.py:2598`; `tools/providers.py:32`). Keeping
that pattern means (a) the stdio entry point keeps working for sessions that prefer it, (b) no new IPC, (c) tests use
`respx` like all other tool tests. Rejected: importing the registry directly from the tool module — it would break in
stdio mode where the tool process is not the backend process; a separate MCP process — contradicts "existing address".
Single-worker only, like `_active_runs` (`main.py:73-76`); Docker runs one uvicorn (`Dockerfile:35`).

State per slot (memory):
```json
{"slot_id":"claude-1","name":"Claude Code 1","lease_id":"uuid|null","client":"claude-code|codex|local|unknown",
 "last_seen_at":"ISO","waiting":false,"in_flight":"question_id|null","queue":["question_id",...]}
```
State per question (memory, also mirrored to the log): `question_id`, `slot_id`, `kind`, `system`, `content`,
`messages` (full, log only), `conversation_id|null`, `source|null`, `posted_at`, `deadline_at`, `state`
(`queued|delivered|answering|answered|expired|cancelled|released`), `delivered_to` (lease), `chunks: [str]`.

Rules:
- `occupied(slot) = waiting or (now - last_seen_at) < heartbeat_seconds or (in_flight and now < its deadline_at)`.
  The third clause exists because a Claude Code session cannot call heartbeat while it is generating a long answer.
- `take(slot_id, lease_id=None, client)`: unknown slot → `slot_not_found`. If `lease_id` matches the current lease →
  `retaken: true`, same lease. Otherwise a new lease is issued and replaces any existing one (`replaced_lease: true`
  when one existed); the previous occupant's next call fails with `lease_mismatch`. Any question in `delivered` or
  `answering` state stays pending and is re-delivered to the next `wait` on the new lease (chunks so far are kept).
- `wait(slot_id, lease_id, timeout_seconds)`: lease must match; set `waiting=true`, refresh `last_seen_at`; if the
  queue has a question (or an unanswered re-deliverable one), return it now and mark `delivered`; otherwise await the
  slot's `asyncio.Event` up to `min(timeout_seconds, wait_max_seconds)`; on return clear `waiting`, refresh
  `last_seen_at`. Returns `{"status":"idle"}` on timeout.
- `answer(question_id, lease_id, text, final)`: lease must match `delivered_to`; state must be `delivered|answering`;
  append chunk; refresh `last_seen_at`; if `final`, set `answered` and resolve the awaiting future. A late answer to an
  `expired|cancelled` question is logged and returns `question_expired` (no exception).
- `heartbeat`, `release`: lease must match. `release` clears the lease; any in-flight question is set `released` and
  its future resolved with the `slot_released` error.
- `post(slot_id, question)`: append, set event. Used by the provider and by "send to session".
- `expire(question_id)`: provider calls on timeout; removes from queue if still `queued`.

### 5.2 Agent-session provider (`backend/providers/agent_session.py`)

`query(model_id, messages, timeout, temperature)` — step by step:
1. `slot_id = model_id.removeprefix("agent-session:")`. Unknown slot → return
   `{"content": null, "error": "slot_not_found", "error_message": "agent-session: unknown slot '<id>'"}`.
2. If not `occupied(slot)` → return **immediately**
   `{"content": null, "error": "slot_not_connected", "error_message": "slot not connected: agent-session:<id> has no session waiting or heartbeating within the last <N> s"}`.
   The text must not contain "timeout"/"timed out", so preflight counts it as a hard failure and blocks the run
   (`model_preflight.py:95-97,135-139`) — this is the brief's "fails immediately" [20-21].
3. Build the question: `system` = contents of all `role == "system"` messages joined by a blank line, or null
   (advisors send one, `backend/advisors.py:161-164`); `content` = content of the **last** `role == "user"` message;
   earlier turns (`council.py:222` builds `history + [user]`) are **not** sent but are stored in the log record's
   `messages`. `kind = "preflight"` when `messages == [{"role":"user","content":"Reply with OK."}]`, else
   `"council"`. The constant is duplicated in the provider (importing `model_preflight` would be circular:
   `model_preflight.py:11` imports `council`, which imports providers); a test pins equality (section 9).
4. `question_id = uuid4`, `deadline_at = now + timeout`; `registry.post(...)`; log `question`.
5. `await asyncio.wait_for(future, timeout)`. `temperature` is ignored.
6. On final answer: `content = "".join(chunks)`; return
   `{"content": content, "usage": {"prompt_tokens": ceil(len(system+content_in)/4), "completion_tokens": ceil(len(content)/4), "total_tokens": sum, "estimated": true}, "error": null}`.
   `attach_cost` normalises this (`costs.py:713-721`); `agent-session` is added to `_SUPPORTED_PROVIDER_PREFIXES`
   (`costs.py:32-48`) and to the known-free rule beside the OAuth prefixes (`costs.py:49`) so cost is `$0`.
7. On timeout: `registry.expire(question_id)`; return
   `{"content": null, "error": "timeout", "error_message": "Request timed out after <N>s — agent-session:<id> did not finish its answer (<k> chars received)"}`.
   Contains "timed out", so preflight treats it as soft (run proceeds) while a real run records a failed model.
8. On `slot_released` (session released mid-question): return
   `{"content": null, "error": "slot_released", "error_message": "slot released: the session left agent-session:<id> before answering"}`.
9. On `asyncio.CancelledError` (client disconnected, `council.py:250-254`): mark the question `cancelled` and re-raise.

Queueing: FIFO per slot. A second question posted while one is in flight waits in the queue; the session's next
`wait` returns it. Nothing is dropped, nothing is merged.

Session expiry mid-question: heartbeat lapse alone does not fail the question (the deadline clause keeps the slot
occupied); the provider waits until `deadline_at`. Only `release` or a retake-with-re-delivery change the outcome.

`get_models()`: one entry per configured slot, no network:
```json
{"id":"agent-session:claude-1","name":"Claude Code 1 [Agent Session] — connected","provider":"Agent Session",
 "source":"agent-session","is_free":true,"occupied":true,"slot_id":"claude-1","client":"claude-code"}
```
(`— empty` when not occupied). `validate_key(api_key)`: ignores the argument, returns
`{"success": true, "message": "Agent sessions need no key. <n> of <m> slots connected."}`. Note `test-provider`
short-circuits on an empty key (`main.py:2363-2364`), so the Settings card reads `GET /api/agent-sessions/slots`
instead (5.6).

Timeout default: `request_timeout()` gains one lookup between the per-provider env var and the global env var: for
`agent-session` it returns `settings.agent_session_timeout_seconds` (default 900). Precedence becomes
`AGENT_SESSION_REQUEST_TIMEOUT` → settings value → `LLM_COUNCIL_REQUEST_TIMEOUT` → 180. Rationale: a 10 tok/s local
model writing 2 000 tokens needs ≥ 200 s plus thinking; 180 s (`timeouts.py:31`) cuts it [24-25]. Preflight still
passes its own 5 s explicitly (`model_preflight.py:112-118`) and the provider honours whatever it is given.

### 5.3 MCP tools (`the_ai_counsel_mcp/tools/agent_session.py`)

One action-based tool, following the repo convention (`docs/mcp/TOOLS.md:3`), named **`agent_session`**. All results
are JSON strings. Errors: `{"status":"error","error":"<code>","message":"<plain text>"}` with codes
`slot_not_found | lease_mismatch | slot_empty | question_not_found | question_expired | already_answered | invalid_args | backend_unreachable`.

| action | input | output (success) | REST route used |
|---|---|---|---|
| `list` | — | `{"slots":[{"slot_id","name","occupied","waiting","client","last_seen_at","pending":int,"in_flight":"qid\|null"}]}` | `GET /api/agent-sessions/slots` |
| `take` | `slot_id` (required), `lease_id?`, `client?` | `{"slot_id","lease_id","retaken":bool,"replaced_lease":bool,"heartbeat_seconds":90,"wait_max_seconds":90}` | `POST /api/agent-sessions/slots/{id}/take` |
| `release` | `slot_id`, `lease_id` | `{"slot_id","released":true}` | `POST .../slots/{id}/release` |
| `wait` | `slot_id`, `lease_id`, `timeout_seconds?` (default 50, max 90) | `{"status":"question","question_id","slot_id","kind":"council\|preflight\|task","system":"str\|null","content":"str","conversation_id":"str\|null","source":{...}\|null,"posted_at","deadline_at"}` or `{"status":"idle","slot_id"}` | `GET .../slots/{id}/wait?lease_id=&timeout=` |
| `answer` | `question_id`, `lease_id`, `text`, `final?` (default true) | `{"question_id","accepted_chars":int,"final":bool,"state":"answering\|answered"}` | `POST /api/agent-sessions/questions/{qid}/answer` |
| `heartbeat` | `slot_id`, `lease_id` | `{"slot_id","last_seen_at","pending":int}` | `POST .../slots/{id}/heartbeat` |
| `log` | `slot_id`, `limit?` (default 50), `since?` (ISO) | `{"slot_id","entries":[...]}` (records as in 5.9) | `GET .../slots/{id}/log` |
| `send` | `slot_id`, `content`, `conversation_id?` | `{"question_id","slot_id","queued_behind":int}` | `POST .../slots/{id}/send` |

REST routes live in `backend/agent_sessions/routes.py` as an `APIRouter` included from `main.py`; no auth (brief
[26]); they are not behind `_require_admin` (`main.py:314-338`). `CouncilClient` gains one method per route
(`client.py`). `/api/health` reports `"tools": 11` (`main.py:707`).

Transport. `create_server()` sets FastMCP `streamable_http_path="/"`, `stateless_http=True`, `json_response=False`
(SDK settings at `.venv/.../fastmcp/server.py:166-168`). `main.py` composes one Starlette sub-app mounted at `/mcp`
whose route table is, in order: `Route("/sse")` and `Mount("/messages/")` taken from the existing SSE app
(`sse_app()`, `server.py:818`), then `Mount("/", streamable_http_app)`. Route order makes `/mcp/sse` still resolve to
SSE while `POST|GET|DELETE /mcp` is streamable HTTP. A FastAPI `lifespan` is added that enters
`_mcp.session_manager.run()` (required once per process, `streamable_http_manager.py:99-105`). Stateless mode is
chosen because each tool call is independent and a session that reconnects after a network blip needs no
`Mcp-Session-Id` bookkeeping; server-to-client notifications are not needed. SSE stays (open decision D3).

Session configuration:
```
claude mcp add --transport http the-ai-counsel http://192.168.1.20:8001/mcp
```
Codex CLI `~/.codex/config.toml` (inferred from Codex documentation; owner to verify against the installed version):
```toml
[mcp_servers.the-ai-counsel]
url = "http://192.168.1.20:8001/mcp"
```
The backend must bind the LAN interface: `start.sh:38` already passes `0.0.0.0`; `python -m backend.main` defaults to
`127.0.0.1` (`main.py:2614`); Docker binds `0.0.0.0` (`Dockerfile:35`). No new port, path or address is ever needed by
a session; slots, leases and questions are all addressed inside tool arguments [18-19].

### 5.4 Streaming pass-through

Chunks arrive through `answer(final=false)`; the registry appends to `chunks`, refreshes `last_seen_at`, writes an
`answer_chunk` log record, and notifies subscribers of that question. The non-streaming caller (the council) is
`query()` awaiting the future, which resolves only on `final=true`; it returns the joined text once, exactly as any
provider does today. A future streaming caller consumes `registry.subscribe(question_id)` (an async iterator of
`{"seq","text","final"}`) or `GET /api/agent-sessions/questions/{qid}/stream`, an SSE route emitting the same
records; the UI's active-run progress (`_active_runs`, `main.py:73-76`) may later show `answering: <k> chars`. On
timeout the partial text is preserved in the log and its length reported in `error_message` (5.2 step 7); the
partial is not returned as `content` because the brief says the caller waits for the whole answer [24].

### 5.5 Skill for the session (`skills/the-ai-counsel-session/SKILL.md`)

Front matter like `skills/the-ai-counsel-api/SKILL.md`. Body, verbatim intent:

```
You are one member of The AI Counsel. Loop until told to stop.
1. agent_session take slot_id=<SLOT> lease_id=<LEASE if you have one> client=<claude-code|codex|local>.
   Keep the returned lease_id. If replaced_lease is true, another session was here; continue anyway.
2. agent_session wait slot_id=<SLOT> lease_id=<LEASE> timeout_seconds=50.
   - status "idle": go to 2 again immediately. Do not summarise, do not ask the user.
   - status "question": go to 3.
3. Answer the question in `content`, obeying `system` if present.
   - kind "preflight": reply exactly "OK" with agent_session answer final=true. Go to 2.
   - kind "council": write the full answer as the model would. If you need more than ~60 s (reading files,
     running commands), call agent_session heartbeat every ~60 s between steps. Send long answers in parts with
     final=false and the last part with final=true. Never invent facts to be faster. Go to 2.
   - kind "task": do the work in the repository (edit, test, commit as the task says). Heartbeat between steps.
     When done, answer final=true with a short report: files changed, tests run, open points. Go to 2.
4. Errors:
   - lease_mismatch: another session took this slot. Do not fight. Run agent_session list, take a different
     empty slot if the user wants, otherwise tell the user and stop.
   - question_expired: your answer came after the deadline. It is logged; do not resend. Go to 2.
   - slot_not_found / backend_unreachable: report once, retry take after 30 s, stop after 5 failures.
5. Stop conditions: the user says stop; you called agent_session release; five consecutive backend errors.
Never change files during a "council" or "preflight" question. Never answer a question you did not receive.
```
Stop conditions and the "no hooks" requirement are met: the loop is ordinary tool calls [27].

### 5.6 UI (minimal)

- `frontend/src/constants/agentSessions.js` (new): `AGENT_SESSION_PREFIX = "agent-session:"`,
  `filterAgentSessionModels(directModels, settings)` = keep prefix matches when
  `settings.enabled_providers["agent-session"] !== false`.
- `CouncilSetup.jsx:26-42`, `AdvisorSetup.jsx:124-140`, `Settings.jsx:1650-1680`: exclude the prefix in the direct
  filter (as OAuth is excluded at `CouncilSetup.jsx:30-32`) and append `filterAgentSessionModels(...)`.
- `SearchableModelSelect.jsx:19-41,53-59`: group label `Agent Sessions (Live)`; option label appends `● connected`
  or `○ empty` from `model.occupied`.
- `councilGridUtils.js:13-29,31-47`: `PROVIDER_CONFIG["agent-session"] = {label: "Session", color, icon}` and the
  prefix row, so grids and Stage tabs render an icon (`AGENTS.md` "Provider Icon Detection").
- Settings → LLM API Keys: new card `AgentSessionSettings.jsx` (no key field): enable toggle
  (`enabled_providers["agent-session"]`), slot table (id, name, connected/empty, last seen; polls
  `GET /api/agent-sessions/slots` every 5 s, pattern as `SubscriptionOAuth.jsx:5`), add/remove/rename slots
  (`agent_session_slots`), heartbeat window and timeout numbers. Saves via `PUT /api/settings`
  (`UpdateSettingsRequest`, `main.py:1702`, three new optional fields).
- `api.js:365-369` area: `getAgentSessionSlots()`, `sendToAgentSession(slotId, content, conversationId)`.

### 5.7 Outcome to code

UI: a "Send to session…" button beside the copy button on the chairman answer (`Stage3.jsx:38,79`) and on the advisor
verdict (`Stage3.jsx:139-140`; verdict shape `{model, content, error, usage, cost}` from `advisors.py:196-215`). It
opens a slot chooser (connected slots first) and calls `POST /api/agent-sessions/slots/{id}/send`:
```json
{"content":"<verdict text>","conversation_id":"<id>","source":{"kind":"council_verdict|advisor_verdict","conversation_id":"<id>","message_index":3,"question":"<user question, first 500 chars>"}}
```
The route wraps it as a `kind: "task"` question whose `content` is the template
`AGENT_SESSION_TASK_TEMPLATE` (constant in `backend/agent_sessions/registry.py`):
"The AI Counsel reached this verdict on: <question>\n\n<verdict>\n\nCarry it out in your repository. Report what you
changed when done." The session receives it from `wait` like any question; `system` is null; no caller awaits the
answer, but its report is logged and shown in the slot log. MCP `send` does the same for agents. This is the brief's
third objective [33-34]: the decision lands in the repository through the session's own tools.

### 5.8 Config and defaults

`backend/settings.py` fields (all with defaults, so old `settings.json` files load unchanged):
```json
{"agent_session_slots":[{"id":"claude-1","name":"Claude Code 1"},{"id":"codex-1","name":"Codex 1"},{"id":"local-1","name":"Local runner 1"}],
 "agent_session_heartbeat_seconds":90,"agent_session_timeout_seconds":900,"agent_session_wait_max_seconds":90}
```
`DEFAULT_ENABLED_PROVIDERS["agent-session"] = false` (`settings.py:47-56`). Slot ids match `^[a-z0-9][a-z0-9-]{0,31}$`,
max 16 slots, normalised on load like presets (`settings.py:238-330`). Bounds: heartbeat 15–600 s; timeout 30–3600 s
(the `timeouts.py:36-37` cap); wait max 5–90 s and never above the heartbeat window, so a waiting session is
always occupied. Env: `AGENT_SESSION_REQUEST_TIMEOUT` (documented in `.env.example`). No auth, no TLS [26].

### 5.9 Persistence

In memory only: slot occupancy (lease, last seen, waiting), queues, in-flight questions and chunks, subscribers. A
backend restart empties every slot; sessions notice on their next call (`lease_mismatch`) and retake (skill 4).

On disk: `data/agent_sessions/<slot-id>.jsonl`, one JSON object per line, appended synchronously by the registry
(`backend/agent_sessions/log.py`), directory beside `data/conversations` (`config.py:12`) and covered by the Docker
volume (`docker-compose.yml:21`). Record kinds: `take`, `release`, `question` (with full `messages`), `delivered`,
`answer_chunk`, `answer` (joined text, chars, seconds), `expired`, `cancelled`, `late_answer`. Every record has
`ts`, `slot_id`, `lease_id`, `question_id|null`. Slot definitions are in `settings.json`. Nothing about slots is
written into conversation files; the stage-1 result already records `model: "agent-session:claude-1"`.

## 6. Sequence walkthroughs

(a) **Preflight to a connected Claude session.** User starts a run with `agent-session:claude-1` selected.
`preflight_models` calls `query_model(..., timeout=5.0)` (`model_preflight.py:112-118`). Provider: slot occupied
(session in `wait`) → post `kind: preflight` question, deadline now+5 s. The session's `wait` returns
`{"status":"question","kind":"preflight","content":"Reply with OK."}`; the model calls `answer(text="OK", final=true)`.
If that lands within 5 s, `query()` returns `{"content":"OK"}` and preflight passes. If it lands later, `query()` has
returned the timeout error (soft: run proceeds, `model_preflight.py:235-241`) and the late `answer` returns
`question_expired`; the log shows `expired` then `late_answer`. If the slot was empty, `slot_not_connected` blocks the
run with the message from `build_preflight_error_message` (`model_preflight.py:250-264`).

(b) **Council with three slots as three models.** Members `agent-session:claude-1`, `agent-session:codex-1`,
`agent-session:local-1`, chairman `github-copilot:gpt-4.1`. Stage 1 posts one question per slot in parallel
(`council.py:245`); each session answers; results stream to the UI as they complete. Stage 2 posts the ranking prompt
(self-contained, `council.py:340-370`) to the same slots; each session sees only that prompt. Stage 3 goes to Copilot.
Cost report shows `$0` for the three slots (known-free rule). The three logs each hold two `question`/`answer` pairs.

(c) **Session silent mid-question.** Stage 1 question delivered to `codex-1`; the session hangs. Heartbeats stop, but
the in-flight clause keeps the slot occupied until `deadline_at` (900 s default). At the deadline `query()` returns
`{"error":"timeout","error_message":"Request timed out after 900s — agent-session:codex-1 did not finish ..."}`. The
council records a failed member (`council.py:263-272`), Stage 1 UI shows "Model Failed to Respond"
(`frontend/src/components/Stage1.jsx:135-138`), Stage 2 excludes it (`council.py:326`), the run continues. If the
session later answers, `question_expired`; the slot is empty once `last_seen_at` ages past 90 s.

(d) **Verdict sent for implementation.** The user clicks "Send to session…" on Stage 3, picks `claude-1`. UI posts to
`/send`; route wraps the template as `kind: task`; `wait` in the Claude session returns it; the skill edits files,
heartbeating between steps, and finishes with `answer(final=true, text="Changed a.py, b.py; tests pass")`. The log
shows the task and the report; the UI slot table shows "last answer 2 min ago". Nothing in the council waited.

(e) **Local-model runner answering in chunks.** A small script (any MCP client) takes `local-1` with `client: local`,
loops on `wait`, and on a question streams tokens from Ollama/LM Studio, calling `answer(final=false)` every ~200
characters and `answer(final=true)` at the end. Each chunk refreshes `last_seen_at`; the provider returns the joined
text when the final chunk lands. At 10 tok/s a 1 500-token answer takes ~150 s, well inside 900 s [25].

## 7. Edge cases and failure modes

| # | Situation | Behaviour |
|---|---|---|
| 1 | Question for a slot with no session | `slot_not_connected` at once; preflight hard-fails the run |
| 2 | Slot occupied by heartbeat only (no wait in progress) | Question queued; delivered on the session's next `wait` |
| 3 | Session in `wait` when question posted | `wait` returns within the event-loop tick [11-12] |
| 4 | Two questions posted while one is in flight | FIFO; each `wait` returns the next; none dropped |
| 5 | Same slot assigned to two advisor personas | Two questions queued; answered sequentially by one session |
| 6 | Heartbeat lapses during a long answer | Slot stays occupied via in-flight clause until deadline |
| 7 | Session crashes after `delivered`, before answer | Question waits for deadline; on retake it is re-delivered |
| 8 | Session retakes with its old lease | `retaken: true`; state unchanged |
| 9 | Another session takes an occupied slot | New lease; old session gets `lease_mismatch`; in-flight question re-delivered to new lease |
| 10 | `answer` with wrong lease | `lease_mismatch`; chunk discarded, logged as rejected |
| 11 | `answer` after deadline | `question_expired`; text kept in log as `late_answer` |
| 12 | `answer(final=true)` twice | Second returns `already_answered` |
| 13 | Empty `text` with `final=true` and no prior chunks | Accepted; `query()` returns `content: ""`; council stores empty response (as any provider) |
| 14 | Session calls `release` mid-question | `slot_released` error to the council immediately |
| 15 | Browser disconnects mid-run | Council cancels tasks (`council.py:250-254`); question marked `cancelled`; late answer → `question_expired` |
| 16 | Backend restarts | All slots empty; leases invalid; sessions retake per skill 4; logs intact on disk |
| 17 | Slot removed in Settings while occupied | Registry drops it; occupant's next call `slot_not_found`; in-flight question `cancelled` |
| 18 | Slot renamed | Id unchanged; only `name` changes; council presets unaffected |
| 19 | Slot id in council preset no longer configured | `query()` → `slot_not_found` (hard preflight failure, clear message) |
| 20 | `wait` timeout above `wait_max_seconds` | Clamped to the max; response says so in `idle` payload |
| 21 | MCP client tool-call timeout shorter than `wait` (inferred risk for Claude Code) | Skill default 50 s; if the client still times out, user lowers `timeout_seconds` |
| 22 | Non-string `content` in messages (documents) | Provider stringifies as council does (`council.py:277-281`) |
| 23 | Preflight timeout 5 s vs a slow session | Soft timeout, run proceeds; late "OK" logged as `late_answer` |
| 24 | Question posted for an occupied slot whose session is in `wait` with a different lease (stale) | Stale `wait` gets `lease_mismatch` on its next call; question waits for the live lease |
| 25 | Log file unwritable | Registry logs a warning and continues in memory (mirrors `storage.py` best-effort style) |
| 26 | `agent-session` disabled in Settings but slot selected via API | Same as other providers: query proceeds; the picker simply hides it |
| 27 | Multi-worker uvicorn | Unsupported; documented, same as `_active_runs` (`main.py:75`) |
| 28 | Streamable HTTP request without lifespan started (tests) | 500 from the SDK; tests must use `with TestClient(app)` |

## 8. Compatibility

Upstream files touched (kept small for future merges): `backend/council.py` (+import, +1 `PROVIDERS` entry);
`backend/providers/timeouts.py` (+settings lookup for `agent-session`); `backend/costs.py` (+prefix, +free rule);
`backend/settings.py` (+4 fields, +1 default toggle, +normaliser); `backend/settings_payload.py` (+4 keys);
`backend/main.py` (+lifespan, +`/mcp` composition, +router include, +3 `UpdateSettingsRequest` fields, health tool
count 11); `the_ai_counsel_mcp/server.py` (+register, +FastMCP settings, instructions prefix list);
`the_ai_counsel_mcp/client.py` (+8 methods); frontend `api.js`, `CouncilSetup.jsx`, `AdvisorSetup.jsx`, `Settings.jsx`,
`SearchableModelSelect.jsx`, `councilGridUtils.js`, `ProviderSettings.jsx` (mount the new card), `Stage3.jsx`; docs
per `docs/DOC-SYNC.md` (`SKILL.md` prefix table `:149-162`, `docs/mcp/TOOLS.md`, `AGENTS.md`, `README.md`,
`CHANGELOG.md` Unreleased, `.env.example`).

New files: `backend/agent_sessions/{__init__,registry,log,routes}.py`, `backend/providers/agent_session.py`,
`the_ai_counsel_mcp/tools/agent_session.py`, `skills/the-ai-counsel-session/SKILL.md`,
`frontend/src/constants/agentSessions.js`, `frontend/src/components/settings/AgentSessionSettings.jsx`, tests below.

Settings migration: none required; pydantic defaults fill missing keys (`settings.py:406`); export/import carry the
new fields through `model_dump` (`settings_payload.py:127-158`). Docker: no image or compose change; `data/` volume
already covers the log directory. `start.sh`: no change. Version bump follows `AGENTS.md` "Versioning Checklist".

Existing tests affected: none should break. `create_server()` tests gain one tool; `/api/health` tool count has no
test today (grep). Adding a FastAPI lifespan is safe for tests already using `with TestClient(app)`
(`test_main_preflight.py:55,71,104`; `test_settings_test_provider.py:24-27`).

## 9. Test specification

Backend (`backend/tests/`), all `pytest-asyncio` auto mode:

| File / test | Setup | Assertion | Section |
|---|---|---|---|
| `test_agent_session_registry.py::test_take_issues_lease_and_retake_keeps_it` | fresh registry, one slot | second take with same lease → `retaken` true, lease equal | 5.1 |
| `…::test_take_replaces_lease_and_old_lease_fails` | take twice without lease | `replaced_lease` true; heartbeat with old lease raises `LeaseMismatch` | 5.1 |
| `…::test_wait_returns_immediately_when_question_posted` | task in `wait(timeout=5)`, then `post` | wait returns < 0.1 s with the question | 5.1, [11-12] |
| `…::test_wait_idle_after_timeout_and_refreshes_last_seen` | `wait(timeout=0.2)` | `idle`; `last_seen_at` updated; `waiting` false | 5.1 |
| `…::test_occupied_by_in_flight_deadline_without_heartbeat` | deliver question, set `last_seen_at` old | `occupied()` true until deadline, false after | 5.1 |
| `…::test_release_resolves_in_flight_with_slot_released` | deliver, release | awaiting future resolves with `slot_released` | 5.1 |
| `…::test_retake_redelivers_undelivered_answer` | deliver to lease A, take by B, wait by B | same `question_id`, chunks preserved | 5.1 |
| `…::test_fifo_order` | post q1,q2 | waits return q1 then q2 | 5.2 |
| `test_agent_session_provider.py::test_empty_slot_returns_slot_not_connected_immediately` | registry empty | dict has `error` truthy, message contains "slot not connected", not "timed out"; elapsed < 0.05 s | 5.2 step 2 |
| `…::test_unknown_slot` | — | `error == "slot_not_found"` | 5.2 |
| `…::test_query_sends_only_last_user_turn_and_system` | messages with history + system | posted question `content` == last user, `system` joined; log record has full `messages` | 5.2 step 3, [23] |
| `…::test_preflight_kind_detected_and_constant_matches` | `[{"role":"user","content":PREFLIGHT_PROMPT}]` | `kind == "preflight"`; provider constant == `model_preflight.PREFLIGHT_PROMPT` | 5.2 |
| `…::test_query_returns_joined_chunks_and_estimated_usage` | fake session answers 3 chunks | `content` joined; `usage.estimated` true; `error` falsy | 5.2 step 6, 5.4 |
| `…::test_query_timeout_error_mentions_timed_out_and_partial` | one non-final chunk, timeout 0.3 | `error == "timeout"`, message has "timed out" and char count; question `expired` | 5.2 step 7 |
| `…::test_query_cancelled_marks_question_cancelled` | cancel the task | question state `cancelled`; later answer → `QuestionExpired` | 5.2 step 9 |
| `…::test_get_models_lists_slots_with_occupied_flag` | one occupied, one empty | ids `agent-session:<id>`, `occupied` flags, `is_free` true | 5.2 |
| `…::test_validate_key_always_ok` | — | `success` true regardless of argument | 5.2 |
| `test_model_preflight.py::test_slot_not_connected_is_hard_failure` (added) | patch `query_model` → slot_not_connected dict | `result.failures` non-empty | 4, 5.2 |
| `test_request_timeouts.py::test_agent_session_default_comes_from_settings` (added) | settings field 900, env unset | `request_timeout("agent-session") == 900`; env var overrides | 5.2 |
| `test_costs.py::test_agent_session_is_free` (added) | `attach_cost("agent-session:x", {...})` | `cost.total_cost == 0`, prefix recognised | 5.2 step 6 |
| `test_council_provider_registry.py::test_agent_session_registered` | — | `PROVIDERS["agent-session"]` is the provider; `get_provider_name_for_model("agent-session:claude-1")` | 4 |
| `test_agent_session_routes.py` (TestClient) | `with TestClient(app)` | each route: 200 shapes from 5.3; wrong lease → 409 `lease_mismatch`; unknown slot → 404; `wait?timeout=200` clamped | 5.3 |
| `…::test_send_wraps_verdict_as_task` | POST `/send` | queued question `kind == "task"`, content contains verdict and question | 5.7 |
| `test_agent_session_log.py` | `monkeypatch` log dir to `tmp_path` | JSONL lines parse; record kinds in order; unwritable dir → warning, no raise | 5.9 |
| `test_settings_agent_sessions.py` | load JSON without new keys | defaults applied; invalid slot ids dropped; wait max clamped ≤ heartbeat | 5.8 |
| `test_mcp_transport.py` | `with TestClient(app)` | `POST /mcp` initialize returns a JSON-RPC result; `GET /mcp/sse` still 200 | 5.3 |

MCP (`the_ai_counsel_mcp/tests/test_tools_agent_session.py`, `respx` on `http://test:8001`): one test per action
asserting the exact JSON of 5.3 and the error mapping (HTTP 409 → `lease_mismatch`, 404 → `slot_not_found`, 410 →
`question_expired`, connection error → `backend_unreachable`); `test_server.py` asserts the `agent_session` tool is
listed.

End-to-end (`backend/tests/test_agent_session_e2e.py`): monkeypatch `CouncilClient.__aenter__` to use
`httpx.ASGITransport(app=backend.main.app)`; build `create_server()`; fake session = coroutine calling
`server.call_tool("agent_session", {...})` in order take → wait → (assert `kind == "preflight"`) answer "OK" → wait →
answer two chunks. Concurrently run `preflight_models(["agent-session:claude-1"], timeout=5)` (expect ok) then
`council.query_model("agent-session:claude-1", messages)` (expect joined content). Then a negative pass: release, run
`query_model` again, expect `slot_not_connected` in < 0.1 s. Covers 6(a), 6(b) partially, 6(e).

## 10. Open decisions for the owner

- **D1 Slot provisioning.** Why: the brief says "take one by id" but not who creates ids. Options: (a) slots configured
  in Settings (defaults `claude-1`, `codex-1`, `local-1`), take only existing ids; (b) take creates a slot when the id
  is new. Recommendation: (a) — the picker and council presets need stable ids before any session connects.
- **D2 Lease id vs last-writer-wins.** Why: the brief allows retake but does not say how two sessions on one slot are
  told apart. Options: (a) opaque lease returned on take, required on later calls; (b) no lease, every call accepted.
  Recommendation: (a); it is not a password, it is how "slot taken by another" becomes detectable at all.
- **D3 Keep SSE at `/mcp/sse`?** Why: brief only requires streamable HTTP. Options: keep both (route composition in
  `main.py`), or drop SSE and simplify to `app.mount("/mcp", streamable_http_app())`. Recommendation: keep for one
  release; `README.md:403`, `docs/mcp/*` and `AGENTS.md` all advertise `/mcp/sse`.
- **D4 Tool naming.** Options: one `agent_session` tool with `action` (repo convention, `TOOLS.md:3`) or eight flat
  tools (`slot_take`, `slot_wait`, …), which read more clearly in a skill. Recommendation: one tool, convention wins;
  the skill text names actions explicitly.
- **D5 Picker label.** Options: "Agent Sessions (Live)", "Sessions", "Coding agents". Provider label in model dicts
  "Agent Session". Recommendation: "Agent Sessions (Live)".
- **D6 Log location and format.** Options: `data/agent_sessions/<slot>.jsonl` (append-only, recommended) or one
  pretty JSON file per slot rewritten on each event (matches `storage.py` style but O(n) per write).
- **D7 Task template.** Fixed constant (recommended for pass 1) or a user-editable prompt in Settings like the stage
  prompts (`settings.py:170-175`).
- **D8 Stealing an occupied slot.** Options: take always succeeds (recommended; a crashed session looks alive for up
  to 90 s and the human is at the new session) or refuse unless `force`.

## 11. Traceability

| Brief line | Text (short) | Design sections |
|---|---|---|
| 9 | Counsel stays the caller; loop unchanged | 2 G1, 4 (dispatch), 5.2 |
| 10 | Claude Code / Codex session is a model | 5.2, 5.3 config, 5.5 |
| 11-12 | Cannot be woken; must be in wait; returns instantly | 5.1 wait rule, 5.5 loop, 7 #3, 9 wait test |
| 13 | Nothing reused from agent-chatroom | 2 N1 (no code imported; all new files listed in 8) |
| 16 | New provider type; appears in picker | 5.2 `get_models`, 5.6, 4 (frontend filter evidence) |
| 17-19 | One slot per session; tools list/take/wait/answer/heartbeat; streamable HTTP, same address | 5.1, 5.3, D3 |
| 20-21 | Occupied definition; empty slot fails immediately; id returned on take | 5.1 rules, 5.2 step 2, 5.3 take, 7 #1 |
| 22 | Preflight sent for real | 5.2 step 3 kind, 6(a), 7 #23 |
| 23 | Only the new question; full log per slot readable | 5.2 step 3, 5.3 log, 5.9 |
| 24-25 | Chunks as they arrive; wait for whole answer; timeout long enough | 5.4, 5.2 timeout default, 6(e) |
| 26 | No password, no encryption, LAN HTTP | 5.3 routes without auth, 5.8 |
| 27 | Skill: take, wait, answer, repeat, heartbeat; no hooks | 5.5 |
| 30-31 | Subscriptions only; Copilot/ChatGPT already; sessions bring Claude | 6(b), 5.2 (no key), cost free rule |
| 32 | Local model in the same council | 6(e), 5.4 |
| 33-34 | Verdict to a chosen session as its next task | 5.7, 6(d), 5.3 `send` |
