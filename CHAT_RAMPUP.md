# FamChat ramp-up guide

This guide summarizes how FamChat works in the **famworker** + **fambot** codebase: protocols, HTTP APIs, end-to-end flow, and improvement opportunities. Primary implementation lives in the **fambot** submodule (`fambot/src/…`).

---

## 1. Basics: protocols and APIs

### Canonical protocol (read this first)

**[`docs/CHAT_PROTOCOL.md`](./CHAT_PROTOCOL.md)** (in **famworker**) is the contract for clients (Cleo, famdash `chat.html`, future iOS). It covers:

- **SSE**: lines `data: <JSON>\n\n`, stream ends with `data: [DONE]\n\n`
- **Event types**: `status`, `token`, `token_reset`, `message`, `error`, `entity`, `shader`, `breadcrumb` / `breadcrumbs`, `budget`, `compaction`, `media`, `screenshot_request`, `heartbeat`, `reflection`, `done`
- **Socket.IO** namespace **`/chat`**: server emits **`msg`** with the **same JSON** as SSE payloads
- Greeting, breadcrumb freshness, compaction/thread titles, Gemini Live audio, error codes, budget narrative

### HTTP APIs (implementation)

Routes are defined in **`fambot/src/fam/fam_service.py`** (`public_router`).

| Method | Path | Purpose |
|--------|------|--------|
| POST | `/chat` | Main chat turn: **SSE** stream, or **JSON ack + background** work when `socket_mode` is true |
| POST | `/chat/thread/init` | Returns signed **`session_id`** (+ `thread_id`, `uid`) for Socket.IO auth |
| GET | `/chat/greeting` | Greeting + initial breadcrumbs |
| GET | `/chat/tools` | Tool metadata for clients |
| GET | `/chat/threads` | Thread list |
| GET | `/chat/threads/{thread_id}` | Message history (+ summary, budget) for that thread |
| GET | `/chat/budget` | Budget snapshot |
| POST | `/chat/live/start`, `/chat/live/context`, `/chat/live/tool_call` | Gemini **Live** (realtime audio / tool bridge) |

**Doc vs code:** `CHAT_PROTOCOL.md` still references **`GET /chat/history?thread_id=`**. The implemented history endpoint is **`GET /chat/threads/{thread_id}`** (`get_thread_messages`). Update the protocol doc when you align them.

### Auth and gating

- **`get_current_user_or_service`**: JWT **or** service key (e.g. from nginx) for internal callers.
- **`for_uid`** on request bodies: service-key flows acting for a specific user.
- **`chat_feature_allowed_for_uid`**: gates FamChat; when off, endpoints return canned alpha responses or minimal streams.

### `POST /chat` request body (`ChatInput`)

Notable fields: **`message`**, **`thread_id`**, **`for_uid`**, **`stream_tokens`**, **`image_data`**, **`socket_mode`**, **`mode`**, **`version`** (`v1` vs **`v2`** flow-compiled agent), **`quick_mode`**, **`disabled_tools`**, **`channel`**.

### Process layout

- **`fambot/src/run_service.py`** runs **`socket_app`** from **`fambot/src/service/__init__.py`**: **Socket.IO ASGI app wrapping the FastAPI `app`** (HTTP + `/chat` WebSocket in one server).
- **`fambot/src/service/service.py`** imports **`fam.fam_service`** and mounts **`public_router`**.

### Agents and tools

- **Registry:** **`fambot/src/agents/agents.py`** — keys **`chat`** → `graph.chat_graph.chat_agent`, **`fast_chat`** → `graph.fast_chat_graph.fast_chat_agent`.
- **Routing:** **`fambot/src/graph/chat_router.py`** (fast vs full). **`version == "v2"`** uses **`fambot/src/graph/flow_compiler.py`** (`get_flow_agent`) with fallback to v1.
- **Tools:** **`fambot/src/fam/fam_tools/chat/__init__.py`** — `CHAT_TOOLS`, `TOOL_METADATA`.

### Shared helpers

- **`fambot/src/fam/chat_utils.py`**: budgets, **`persist_chat_message`**, **`ChatErrorCode` / `chat_error`**, compaction helpers, UID resolution, etc.

### Not FamChat

- **`fambot/src/service/leah_socket.py`**: Socket.IO namespace **`/leah`**, user-supplied Anthropic key — separate from **`/chat`**.

---

## 2. End-to-end flow (typical web turn)

**Assumptions:** JWT auth, **`socket_mode: false`**, SSE response.

1. Client sends **`POST /chat`** with `message` and optional `thread_id`, `image_data`, flags.
2. **`chat_endpoint`** resolves **uid**, checks **`chat_feature_allowed_for_uid`**.
3. **Parallel preparation:** budget, **thread history** from famgraph (e.g. `get_path_data_async` on chat paths), timezone, and **router** choosing **`fast_chat`** vs **`chat`** (unless v2 flow agent is selected).
4. User text becomes a LangChain **`HumanMessage`** (multimodal if image).
5. User message persistence may run in a **background task** so streaming is not blocked.
6. **LangGraph** agent runs with **`astream_events`**; handler maps events to **SSE** (`token`, `status`, `entity`, …).
7. **Tools** run during the turn; some emit **`entity`** events for rich UI.
8. Handler emits final **`message`**, **`budget`**, optional breadcrumbs, then **`[DONE]`**.
9. Assistant text persisted via **`persist_chat_message`** in **`chat_utils`**.

**`socket_mode: true`:** HTTP returns quickly (e.g. processing); a **background task** runs the same pipeline and **`emit_to_thread`** in **`fambot/src/service/socket_service.py`** emits **`msg`** to room **`thread:{thread_id}`**. Client uses **`POST /chat/thread/init`** then connects to **`/chat`** with **`uid`**, **`session_id`**, **`thread_id`**.

**Threads UI:** **`GET /chat/threads`**, then **`GET /chat/threads/{thread_id}`** for messages.

### Sequence (conceptual)

```text
Client → POST /chat
  → uid + feature gate
  → budget + history + timezone + router (parallel)
  → HumanMessage (+ optional image)
  → persist user (async task)
  → agent.astream_events → SSE (or socket worker → emit_to_thread)
  → persist assistant
  → final message + budget + [DONE]
```

---

## 3. Improvement opportunities

Useful themes for owning the feature (pick 1–2).

### Documentation and contract

- Align **`CHAT_PROTOCOL.md`** with **actual routes** (`/chat/threads/{thread_id}` vs documented `/chat/history`).
- Align **budget table** in the protocol doc with **`chat_utils.py`** (`THREAD_BUDGET_USD`, `DAILY_BUDGET_USD`, `MONTHLY_BUDGET_USD`, etc.).
- Document **`ChatInput`** fields clients rely on: **`version`**, **`quick_mode`**, **`disabled_tools`**, **`channel`**.

### Tests and quality

- Add **unit tests** for `chat_utils` (error dict shape, budget helpers, pure functions).
- Add **integration** or contract tests for **SSE event ordering** / critical payloads (mock graph + agent where possible).

### Code structure

- **`fam_service.py` is very large**; extracting a small module (e.g. SSE event mapping or “one turn” orchestration) improves reviewability without changing behavior.
- **SSE path vs `socket_mode` path** duplicate logic; a **shared internal API** for “emit one logical client event” reduces drift.

### Performance

- Audit **sync** LLM or blocking I/O inside **`async def`** chat paths; prefer **`ainvoke`** or **`asyncio.to_thread`** where libraries are sync-only.

### Observability

- Structured metrics or logs for **history load timeouts**, **router outcomes**, and **tool failure rates** to guide tuning.

### Developer experience

- Short **“FamChat local dev”** note: env vars, **`SKIP_KEY_UNSAFE`**, service key headers, how famdash points at the service.

### Live audio (later track)

- **`/chat/live/*`** is a separate stack (Gemini WebSocket, PCM). Reasonable **phase two** after core **`POST /chat`** is understood.

---

## Suggested onboarding order

1. Read **`CHAT_PROTOCOL.md`**, then trace **`chat_endpoint`** in **`fam_service.py`** (search `socket_mode`, `version`, `_chat_stream`).
2. Run **`fambot`** via **`python src/run_service.py`**, use famdash **`scripts/famdash-dashboard/js/chat.js`** / **`chat.html`** and inspect **SSE** in browser devtools.
3. Land a **small doc fix** or **`chat_utils` unit test** to learn the **fambot** submodule PR workflow vs **famworker**.

---

## Exporting to Google Docs

This file is plain Markdown. To get a Google Doc:

1. In Google Drive: **New → File upload** → choose **`docs/CHAT_RAMPUP.md`**, then **Open with → Google Docs**,  
   **or**
2. Open a new Google Doc and **paste** the contents of this file (headings and tables usually convert reasonably).

There is no automated push from this environment into your Google account.
