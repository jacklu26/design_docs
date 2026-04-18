# Fambot Chat Protocol

This document describes the chat protocol between the Fambot backend (`fam_service.py`) and frontend clients (Cleo SvelteKit app, chat.html dashboard, future iOS app). It covers both the SSE streaming protocol and the Socket.IO real-time connection.

## Architecture Overview

```
Client (Cleo / iOS / chat.html)
  |
  |-- POST /chat          --> SSE stream (primary)
  |-- Socket.IO /chat     --> Real-time events (parallel)
  |-- GET  /chat/greeting  --> Initial greeting + breadcrumbs
  |-- GET  /chat/threads   --> Thread list
  |-- GET  /chat/history   --> Thread message history
  |-- POST /chat/live/start --> WebSocket config for Gemini Live Audio
```

## SSE Streaming Protocol

### Request

```
POST /chat
Content-Type: application/json
Authorization: Bearer <JWT>

{
  "message": "string",
  "thread_id": "uuid | null",
  "stream_tokens": true,
  "image_data": "base64 | data URI | URL | null",
  "socket_mode": false
}
```

### Response (SSE stream)

Each line is `data: <JSON>\n\n`. The stream ends with `data: [DONE]\n\n`.

### Event Types

| Type | Fields | Description |
|------|--------|-------------|
| `status` | `content: string\|null`, `tool?: string` | Status updates ("Thinking...", "Searching emails..."). `null` clears status. |
| `token` | `content: string` | Streamed token from the LLM. Accumulate to build the response. |
| `token_reset` | _(none)_ | Reset accumulated tokens. Sent when the model re-invokes after tool calls. |
| `message` | `role: "ai"`, `content: string`, `sender: "fambot"` | Final complete AI response. Always sent once. |
| `error` | `code: string`, `reason: string`, `error_id: string`, `debug_url?: string` | Error with correlation ID. Codes: CHAT-001 (model), CHAT-010 (stream), CHAT-099 (unknown). |
| `entity` | `entity_type: string`, `node_path: string`, `data: object` | Rich entity data for inline rendering (action, event, contact, etc.). |
| `shader` | `params: object` | Fractal shader parameters for ambient background. |
| `breadcrumb` | `text: string` | Single suggestion, streamed one at a time after response. |
| `breadcrumbs` | `suggestions: string[]` | Batch of suggestions (legacy, from greeting or persisted messages). |
| `budget` | `thread_cost`, `thread_budget`, `thread_remaining`, `monthly_cost`, `monthly_budget`, `monthly_remaining`, `blocked` | Budget status after response. |
| `compaction` | `summary: string` | Thread was compacted (internal, should not be shown to user). |
| `media` | `url: string` | Media attachment from bot. |
| `screenshot_request` | `reason?: string` | Backend requests a DOM screenshot. Client captures and sends back as image. |
| `heartbeat` | _(none)_ | Keepalive every 10s. Ignore. |
| `reflection` | `content: string` | Internal engagement analysis. Debug only. |
| `done` | _(none)_ | Sent as literal `[DONE]` string, not JSON. Signals stream end. |

### Event Ordering

```
1. status: "Processing..."
2. status: "Thinking..."              (on model start)
3. token, token, token...             (streamed response)
4. status: "Searching emails..."      (on tool start)
5. status: null                       (on tool end)
6. entity: {...}                      (if tool created/viewed entities)
7. token_reset                        (model re-invokes after tools)
8. status: "Thinking..."
9. token, token, token...             (final response)
10. message: {role: "ai", content}    (final complete message)
11. breadcrumb: {text: "..."}         (x3, streamed one at a time)
12. budget: {...}
13. [DONE]
```

### Entity Event Details

Entity events are emitted by tools (`InsertEntityItem`, `ViewEntityItem`) when they create or retrieve entities. They arrive during the tool execution phase (between `status: tool_start` and `status: null`).

```json
{
  "type": "entity",
  "entity_type": "action",
  "node_path": "root.actions",
  "data": {
    "uuid": "abc-123",
    "name": "Sign permission slip",
    "description": "Sage's field trip on Friday",
    "start_time": "2026-03-27",
    "status": {"done": false},
    "ctas": [{"cta": "View form", "url": "https://..."}]
  }
}
```

**Entity types and their key fields:**

| Entity Type | Key Fields |
|------------|------------|
| `action` | `uuid`, `name`, `description`, `start_time`, `deadline`, `status`, `ctas`, `action_owner` |
| `family_event` | `uuid`, `name`, `description`, `location`, `start_time`, `end_time`, `all_day`, `status` |
| `calendar_event` | `uuid`, `name`, `summary`, `location`, `starts_at`, `ends_at` |
| `contact` | `uuid`, `name`, `phone`, `email`, `role` |

**Backwards compatibility:** Entity events are additive. The AI's text response always contains a human-readable summary. Clients that don't handle `entity` events still get a working experience via the text.

## Socket.IO Protocol

### Connection

```
Namespace: /chat
Auth: { uid: string, session_id: string, thread_id: string }
```

Session credentials are obtained via `POST /chat/thread/init`.

### Events

| Direction | Event | Payload |
|-----------|-------|---------|
| Server -> Client | `msg` | Same JSON as SSE events above |
| Server -> Client | `log` | Log entry for debugging |
| Server -> Client | `log_history` | Batch of log entries |

Socket.IO events use the same `type` field as SSE events. The `msg` event wraps the same JSON payload.

## Greeting & Breadcrumbs

```
GET /chat/greeting
Authorization: Bearer <JWT>

Response:
{
  "greeting": "Hi there! How can I help your family thrive today?",
  "breadcrumbs": ["What's on the calendar today?", "Check backpack items", "Decode latest newsletter"]
}
```

Breadcrumbs are context-aware (generated from the conversation) and limited to 3. After each chat response, fresh breadcrumbs are streamed individually via `breadcrumb` events.

**Freshness model:** Breadcrumbs are optimistic — show once if fresh. Clients track freshness:
- Mark fresh when new breadcrumbs arrive
- Auto-expire after 30 seconds of visibility
- Expire immediately when user starts typing
- Never persist stale breadcrumbs between responses

## Thread Management

Thread titles are auto-generated during compaction (when a thread exceeds 40 messages). A lightweight LLM call generates a 6-word title from the conversation summary. The client should refresh the thread list after each response (`handleDone`) to pick up title updates.

### List threads
```
GET /chat/threads -> { threads: [{ thread_id, title, last_message_at, status }] }
```

### Load history
```
GET /chat/history?thread_id=xxx -> { messages: [{ role, content, breadcrumbs? }], summary?, budget? }
```

### Initialize thread (for Socket.IO auth)
```
POST /chat/thread/init { thread_id } -> { session_id, uid, thread_id }
```

## Live Audio (Gemini Live API)

### Start session
```
POST /chat/live/start
{ "model": "gemini-2.5-flash-native-audio-preview-12-2025", "voice": "Puck" }

Response:
{
  "ws_url": "wss://generativelanguage.googleapis.com/ws/...",
  "setup": { model, generation_config, system_instruction, tools },
  "uid": "..."
}
```

The client connects directly to the Gemini WebSocket and sends `{ setup: response.setup }` as the first message.

### Audio format
- **Input:** 16-bit PCM, 16kHz mono, base64-encoded
- **Output:** 16-bit PCM, 24kHz mono, base64-encoded (in `inlineData.data`)

### Message format (client -> Gemini)
```json
{
  "realtimeInput": {
    "mediaChunks": [{ "mimeType": "audio/pcm;rate=16000", "data": "<base64>" }]
  }
}
```

### Message format (Gemini -> client)
```json
{
  "serverContent": {
    "modelTurn": {
      "parts": [
        { "text": "Hello!" },
        { "inlineData": { "mimeType": "audio/pcm;rate=24000", "data": "<base64>" } }
      ]
    },
    "inputTranscription": { "text": "user said this" },
    "outputTranscription": { "text": "model said this" },
    "turnComplete": true
  }
}
```

## Error Codes

| Code | Meaning |
|------|---------|
| `CHAT-001` | Model API failed |
| `CHAT-002` | Message persistence failed |
| `CHAT-003` | Socket.IO connection failed |
| `CHAT-004` | History loading failed |
| `CHAT-005` | Perception/breadcrumb generation failed |
| `CHAT-006` | Budget check failed |
| `CHAT-010` | Stream/astream_events error |
| `CHAT-099` | Unknown error |

## Budget Limits

| Limit | Value |
|-------|-------|
| Thread budget | $5.00 |
| Monthly budget | $40.00 |
| Thread stale timeout | 15 minutes |
| Compaction threshold | 40 messages |
| Safety timeout (client) | 5 minutes |

## Implementation Notes for iOS

1. **SSE parsing:** Each line starts with `data: `. Parse JSON from the payload after `data: `. The `[DONE]` marker is a literal string, not JSON.
2. **Token accumulation:** Accumulate `token` events into a string. On `token_reset`, clear and start fresh. The final `message` event contains the authoritative response.
3. **Entity rendering:** Entity events arrive during tool execution. Attach them to the current streaming message. Render as cards based on `entity_type`.
4. **Breadcrumbs:** After `done`, up to 3 `breadcrumb` events arrive individually. Animate them in gracefully.
5. **Connection status:** Socket.IO provides real-time connection state. SSE is request-response.
6. **Image upload:** Send as base64 data URI in `image_data`. Max 20MB. Images are compressed client-side to 1600px max dimension.
7. **Thread history:** On thread switch, load history via GET endpoint. Messages have `role: "human"|"ai"`.
