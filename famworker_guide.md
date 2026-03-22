# Famworker Repository Guide

A step-by-step walkthrough of the famworker codebase — what it does, how data flows, and where everything lives.

---

## Table of Contents

1. [What Is Fambot?](#1-what-is-fambot)
2. [Repository Layout](#2-repository-layout)
3. [The Two Main Services](#3-the-two-main-services)
4. [Step 1: Chronos — The Scheduler](#4-step-1-chronos--the-scheduler)
5. [Step 2: Genesis — The Per-User Orchestrator](#5-step-2-genesis--the-per-user-orchestrator)
6. [Step 3: Paging — The Data Processor](#6-step-3-paging--the-data-processor)
7. [The Full Email Pipeline](#7-the-full-email-pipeline)
8. [The Calendar Pipeline](#8-the-calendar-pipeline)
9. [The Daily Digest](#9-the-daily-digest)
10. [famgraph.json — The Central Configuration](#10-famgraphjson--the-central-configuration)
11. [Nodes: The Building Blocks](#11-nodes-the-building-blocks)
12. [Policies and Pathways: How Data Flows Between Nodes](#12-policies-and-pathways-how-data-flows-between-nodes)
13. [The Schema: Database Table Definitions](#13-the-schema-database-table-definitions)
14. [GraphEngine: Bridging Config to Live Data](#14-graphengine-bridging-config-to-live-data)
15. [The FastAPI Service and Chat](#15-the-fastapi-service-and-chat)
16. [Profiles: Family Member Data](#16-profiles-family-member-data)
17. [Storage: Postgres, Redis, ClickHouse](#17-storage-postgres-redis-clickhouse)
18. [Cost Tracking and Observability](#18-cost-tracking-and-observability)
19. [Key File Reference](#19-key-file-reference)

---

## 1. What Is Fambot?

Fambot is a family AI assistant. It connects to a family's Gmail and Google Calendar, uses LLMs to understand what's going on, and produces organized outputs:

- **Actions to Take** — Tasks with due dates (e.g. "Sign permission slip by Friday")
- **On the Horizon** — Upcoming tasks with future due dates
- **Key Dates to Add** — Calendar events extracted from newsletters and emails
- **Good to Know** — Insights distilled from school/community newsletters
- **Key Contacts** — Contact information (teachers, coaches, etc.)
- **Backpack Check** — Daily checklist of what each child needs (uniforms, gear, etc.)
- **Schedule** — Today's calendar events

Users receive a daily digest email with this information, and can also chat with Fambot to ask questions about their family's schedule and data.

---

## 2. Repository Layout

```
famworker/
├── src/                    # Temporal workers and workflows (the data pipeline)
│   ├── workflows/          #   Chronos, Genesis, Paging, DigestScheduler
│   ├── activities/         #   Email, calendar, agent, partition activities
│   ├── models/             #   Request/response dataclasses
│   ├── clients/            #   External service clients
│   ├── run_worker.py       #   Main worker entry point
│   ├── run_activity_worker.py
│   ├── run_sync_worker.py
│   └── run_memory_activity_worker.py
│
├── fambot/                 # AI agent service (FastAPI + chat)
│   └── src/
│       ├── fam/            #   Core business logic, tools, activities
│       │   ├── fam_service.py    # FastAPI routes (/fam, /chat, /search)
│       │   ├── fam_tools/        # LLM tools for agents
│       │   └── observability/    # Cost tracking, ClickHouse
│       ├── memory/         #   Graph-based memory system
│       │   ├── famgraph.json     # THE central configuration file (~9000 lines)
│       │   ├── models.py         # Node, Entity, Graph classes
│       │   ├── policy.py         # Policy, Pathway, GroundFlow classes
│       │   ├── injected_models.py # GraphEngine (the runtime bridge)
│       │   └── profile.py        # Family profiles
│       ├── graph/          #   LangGraph agent definitions (chat, cleo)
│       ├── agents/         #   Agent registry
│       ├── db/sql/         #   Database layer (SQLAlchemy, Alembic)
│       └── run_service.py  #   FastAPI entry point
│
├── docs/                   # MkDocs documentation
├── scripts/                # Deploy, connect, dashboard scripts
├── helm/                   # Kubernetes Helm charts
├── terraform/              # GCP infrastructure as code
├── tests/                  # Root-level tests
├── corrections/            # Data correction configs
├── queries/                # Ad-hoc SQL/CSV exports
│
├── compose.yaml            # Local Docker Compose
├── client.py               # CLI client for workers
├── CLAUDE.md               # AI guidance for the repo
└── WORKFLOW_ARCHITECTURE.md # Temporal workflow documentation
```

---

## 3. The Two Main Services

### Service A: Temporal Workers (`src/`)

The **data pipeline**. Four separate workers run as containers:

| Worker | Entry Point | Task Queue | Role |
|--------|-------------|------------|------|
| Main Worker | [`src/run_worker.py`](https://github.com/fambots/famworker/blob/main/src/run_worker.py) | `genesis-task-queue` | Runs Chronos, Genesis, Paging workflows |
| Activity Worker | [`src/run_activity_worker.py`](https://github.com/fambots/famworker/blob/main/src/run_activity_worker.py) | `activity-task-queue` | Runs email, calendar, agent, partition activities |
| Memory Worker | [`src/run_memory_activity_worker.py`](https://github.com/fambots/famworker/blob/main/src/run_memory_activity_worker.py) | `memory-activity-task-queue` | Graph queries, memory storage |
| Sync Worker | [`src/run_sync_worker.py`](https://github.com/fambots/famworker/blob/main/src/run_sync_worker.py) | `sync-task-queue` | Billing sync, onboarding, Customer.io |

### Service B: Fambot FastAPI (`fambot/`)

The **API layer**. A single FastAPI service that:

- Serves data to the frontend via `GET /fam/{path}`
- Handles chat via `POST /chat`
- Manages OAuth, integrations, and admin operations

Both services share the same `fambot/src/` Python code (models, GraphEngine, tools) — the Temporal workers import fambot modules directly rather than calling the API over HTTP.

---

## 4. Step 1: Chronos — The Scheduler

**File:** [`src/workflows/chronos.py`](https://github.com/fambots/famworker/blob/main/src/workflows/chronos.py)

Chronos is a Temporal scheduled workflow that runs **every 60 seconds**. Its job is simple: pick the next user who needs processing and start a Genesis workflow for them.

```
Every 60 seconds:
  1. Is there a brand-new user who hasn't been scanned yet?
     → Yes: Start Genesis for that user. Done.
  2. Are we already running too many Genesis workflows? (max 2 concurrent)
     → Yes: Skip this cycle. Done.
  3. Find the user whose scan is most overdue.
     → Start Genesis for that user. Done.
```

This is configured as a Temporal Schedule in [`run_worker.py`](https://github.com/fambots/famworker/blob/main/src/run_worker.py):

```python
await client.create_schedule(
    "chronos",
    schedule=Schedule(
        action=ScheduleActionStartWorkflow(ChronosWorkflow.run, ...),
        spec=ScheduleSpec(intervals=[ScheduleIntervalSpec(every=timedelta(seconds=60))]),
        policy=SchedulePolicy(overlap=ScheduleOverlapPolicy.SKIP),
    ),
)
```

**Key concept:** Chronos does NOT process data itself. It only decides WHO gets processed next.

---

## 5. Step 2: Genesis — The Per-User Orchestrator

**File:** [`src/workflows/genesis.py`](https://github.com/fambots/famworker/blob/main/src/workflows/genesis.py)

Genesis is the per-user orchestrator. When Chronos picks a user, Genesis runs the full processing pipeline for that user.

```
Genesis(uid):
  1. famspec_activity(uid)
     → Loads famgraph.json, builds Pathways from all active policies
     → Returns pathways grouped by phase

  2. For each phase (in order):
     Phase 0 (external): Start PagingWorkflows in PARALLEL
       - root.emails (Gmail)
       - root.events (Google Calendar)

     Phase 1+ (internal): Start PagingWorkflows SEQUENTIALLY
       - root.family_emails (depends on root.emails)
       - root.proto_actions (depends on root.family_emails)
       - root.actions (depends on root.proto_actions)
       - ... and so on

  3. Wait for all PagingWorkflows to signal completion.
```

**Key concept:** Genesis doesn't do the actual data processing either. It orchestrates the order in which PagingWorkflows run, respecting data dependencies between nodes.

---

## 6. Step 3: Paging — The Data Processor

**File:** [`src/workflows/paging.py`](https://github.com/fambots/famworker/blob/main/src/workflows/paging.py)

Paging is where the actual work happens. Each PagingWorkflow handles one pathway (one node + one policy). There are three patterns:

### Pattern A: External Source (Gmail, Calendar)

```
PagingWorkflow(root.emails):
  1. email_activity() → Call Gmail API, page through emails
  2. Store results in Redis (too large for Temporal)
  3. Signal downstream PagingWorkflows with the Redis key
```

### Pattern B: On-Start (initialization)

```
PagingWorkflow(on_start):
  1. Send empty batch to targets
  2. Signal Genesis: "I'm done"
```

### Pattern C: Internal Source (node-to-node processing)

```
PagingWorkflow(root.family_emails):
  1. Receive signal from upstream (root.emails) with data_key
  2. partition_activity() → Split data into batches
  3. agent_activity() → Run LLM to extract/transform data
  4. Store results in Postgres
  5. Signal downstream PagingWorkflows with results
  6. When all batches done: Signal Genesis: "I'm done"
```

**Key concept:** Data stays in Redis; PagingWorkflows only pass Redis keys to each other.

---

## 7. The Full Email Pipeline

This is the most important data flow. Follow it step by step:

```
Step 1: Gmail API
  │  email_activity() pages through the user's inbox
  ▼
Step 2: root.emails (entity: email)
  │  Raw email records stored in Postgres
  │  agent_activity() filters for family-relevant emails
  ▼
Step 3: root.family_emails (entity: family_email)
  │  Only school newsletters, activity updates, etc.
  │  agent_activity() extracts structured data
  ▼
Step 4: "Proto" entities (draft extractions)
  │
  ├─► root.proto_events       → candidate calendar events
  ├─► root.proto_actions       → candidate action items
  ├─► root.proto_contacts      → candidate contacts
  ├─► root.proto_insights      → candidate "good to know" items
  └─► root.proto_backpack_items → candidate backpack checklist items
  │
  │  agent_activity() refines, deduplicates, enriches
  ▼
Step 5: Final entities
  │
  ├─► root.family_events      → Key Dates to Add
  ├─► root.actions             → Actions to Take / On the Horizon
  ├─► root.contacts            → Key Contacts
  ├─► root.insights            → Good to Know
  └─► root.backpack_items      → Backpack Check
```

Each arrow is a PagingWorkflow running agent_activity (an LLM call). The "proto" entities are an intermediate stage — the LLM first extracts candidates, then a second pass refines them.

---

## 8. The Calendar Pipeline

Simpler than email — calendar events are already structured:

```
Google Calendar API
  │  calendar_activity() fetches events
  ▼
root.events (entity: event)
  │  Raw calendar events (starts_at, ends_at, summary, location)
  │
  ├─► root.family_calendar_events  → Family events from calendar
  └─► root.proto_backpack_items    → Backpack items derived from events
```

Calendar events also feed into `root.google_events`, which is an "external entity" — queried live from the Google Calendar API rather than stored in Postgres.

---

## 9. The Daily Digest

**File:** [`src/workflows/digest_scheduler.py`](https://github.com/fambots/famworker/blob/main/src/workflows/digest_scheduler.py)

The DigestSchedulerWorkflow runs **every 15 minutes** and handles sending the daily digest email:

```
Every 15 minutes:
  1. get_eligible_users_activity()
     → Checks: Is it the right time in the user's timezone?
     → Checks: Does the user's day-of-week preference allow sending today?
     → Checks: DDIS score >= 40? PCS score >= 80?
     → Checks: Haven't already sent today?

  2. For each eligible user:
     → send_digest_notification_activity(uid, channel="email")
     → Uses Customer.io to send the email with teaser content

Teaser content is pre-computed during Genesis:
  → precompute_digest_teaser_activity() calls the famcrawl API
  → Stored in the digest_teaser table
  → Retrieved at send time
```

**DDIS** (Daily Digest Interest Score) and **PCS** (Profile Completeness Score) are scoring mechanisms that determine if a user has enough data to make a useful digest.

---

## 10. famgraph.json — The Central Configuration

**File:** [`fambot/src/memory/famgraph.json`](https://github.com/fambots/famworker/blob/main/fambot/src/memory/famgraph.json) (~9000 lines)

This is the single most important file in the repo. It defines three things:

### Section 1: `logging_config` (lines 2-84)

Observability settings:

- Which event streams to log (LLM calls, activity executions, workflows)
- ClickHouse buffer configuration (batch size, flush interval, retry policy)
- LLM pricing tables (cost per million tokens for each model)

### Section 2: `graph` (lines 86-3707)

The node tree — defines every data category, how data flows between them, and the policies that drive processing. This is what `famspec_activity` reads to build pathways for Genesis.

### Section 3: `schema` (lines 3708-8919)

Database table definitions — column types, constraints, inheritance, indexes. This is what `sync_spec_to_db.py` reads to generate `generated_schema.py` (the SQLAlchemy ORM).

---

## 11. Nodes: The Building Blocks

A **node** is a location in the graph hierarchy. Each node typically maps to a database table.

### Example: The `events` node

```json
"events": {
    "entity": "event",
    "description": "Events map one-to-one with what's happening in someone's life.",
    "policies": [
        {
            "name": "Event Policy",
            "active": true,
            "traits": {
                "flows": ["calendar_activity"],
                "sources": ["external"],
                "fields": ["name", "description", "starts_at", "ends_at", "location"]
            }
        }
    ]
}
```

This tells the system:
- This node stores `event` records in the `event` database table
- It has one policy that pulls data from `external` (Google Calendar API)
- The policy uses `calendar_activity` as its processing activity
- It extracts these five fields

### Types of Nodes

| Type | Examples | Characteristics |
|------|----------|-----------------|
| **Entity nodes** | `root.emails`, `root.actions`, `root.events` | Have an `entity` and `policies`; store data in Postgres |
| **View nodes** | `feed_v2`, `newsletters` | Composed views with sub-paths, filters, and sorters; aggregate data from other nodes for the frontend |
| **Profile nodes** | `root.profiles.school`, `root.profiles.activity` | Define family member profile schemas (identity, schedule, gear, etc.) |
| **External nodes** | `root.google_events` | Query live APIs instead of Postgres |

### Node Catalog (major nodes)

```
root.emails                → Raw emails from Gmail
root.family_emails         → Family-relevant emails (filtered by LLM)
root.email_summaries       → Newsletter summaries
root.events                → Raw calendar events
root.family_events         → Enriched family events
root.actions               → Action items with due dates
root.contacts              → People's contact info
root.insights              → "Good to Know" information
root.backpack_items        → Daily backpack checklists
root.family_activities     → Ongoing activities (sports, clubs)
root.actor_roles           → Who does what in activities
root.fam_profiles          → Profile facts with composition
root.subscribers           → Users
root.family.children       → Children in the family
root.family.adults         → Adults in the family
root.calendars             → Calendar preferences
root.chat_threads          → Chat conversation threads
root.chat_messages         → Chat messages
root.weather               → Weather data
root.notification_settings → User notification preferences
root.digest_teasers        → Pre-computed digest content
```

---

## 12. Policies and Pathways: How Data Flows Between Nodes

### Policies

A **policy** is attached to a node and defines a data processing pipeline. Key fields:

```
Policy:
  name: "Family Emails From Emails"
  active: true                          ← Kill switch (set to false to disable)
  traits:
    flows: ["agent_activity"]           ← Which activity to run
    sources: ["root.emails"]            ← Where data comes from
    fields: ["subject", "sender", ...]  ← What fields to extract
  model: "gemini-2.5-flash"            ← Which LLM model to use
  batch_size: 5                         ← How many records per LLM call
  categories: ["root.categories.school"] ← Category filters
```

A node can have multiple policies (e.g. different extraction strategies).

### From Policy to Pathway

When Genesis runs `famspec_activity`, it:

1. Iterates all nodes in the graph
2. For each node, iterates its policies
3. Skips any policy where `active = false`
4. Converts each active policy into a **Pathway** object
5. Groups pathways by phase (external first, then internal in dependency order)

A Pathway is the runtime representation that Paging workflows use. It contains everything needed to run: source, activity, fields, model, batch size, filters, etc.

### Pipeline Examples

Here are some of the data pipelines defined by policies in famgraph.json:

```
external → calendar_activity → root.events
external → email_activity → root.emails
root.emails → agent_activity → root.family_emails
root.family_emails → agent_activity → root.proto_actions
root.proto_actions → agent_activity → root.actions
root.email_summaries_unranked → agent_activity → root.proto_events
root.proto_events → agent_activity → root.family_events
root.events → agent_activity → root.proto_backpack_items
root.proto_backpack_items → agent_activity → root.backpack_items
root.family_emails → agent_activity → root.family_activities
root.family_activities → agent_activity → root.actor_roles
```

---

## 13. The Schema: Database Table Definitions

The `schema` section of [`famgraph.json`](https://github.com/fambots/famworker/blob/main/fambot/src/memory/famgraph.json) defines ~92 database tables. Tables use **inheritance** from four base schemas:

### Base Schemas

**`system_base`** — Every table gets these columns:
- `created_at`, `updated_at` — Timestamps
- `final` — Boolean flag (is this record finalized?)
- `node_path`, `policy_name`, `pathway_id` — Which pipeline created this record
- `scan_sequence_number` — Which scan cycle created this
- `git_commit` — Code version that created this

**`gist_base`** — For tables that support semantic search:
- `gist` — Text summary of the record
- `gist_embedding` — Vector embedding (for similarity search)
- `gist_metadata`, `gist_model` — Embedding metadata

**`entity_base`** — For user-owned records:
- `uid` — User ID
- `source_id` — Original source identifier

**`belief`** — For multi-source records (data from multiple emails/events):
- `sources`, `sources_cnt` — Source tracking
- `attributes`, `profile_params` — Structured data from sources

### Example Entity Definition

```json
"action": {
    "_extends_from": {
        "bases": ["system_base", "gist_base", "belief"]
    },
    "_uniq_constraints": {
        "_gist_excl": { ... }
    },
    "title": { "type": "string", "description": "Action title" },
    "deadline": { "type": "timestamp", "description": "When this must be done" },
    "startline": { "type": "timestamp", "description": "When to start showing this" },
    "assignee": { "type": "string" },
    "priority": { "type": "string" },
    "status": { "type": "string", "default": "'pending'" }
}
```

The action table inherits ~30+ columns from its three bases, plus its own columns. The `_extends_from` mechanism is resolved by [`schema_utils.extend_from_base()`](https://github.com/fambots/famworker/blob/main/fambot/src/memory/schema_utils.py) at schema generation time.

### Schema → Database Flow

```
famgraph.json (schema section)
  ↓ sync_spec_to_db.py
generated_schema.py (SQLAlchemy ORM — do NOT edit manually)
  ↓ alembic revision --autogenerate
Alembic migration file
  ↓ alembic upgrade head
PostgreSQL tables
```

---

## 14. GraphEngine: Bridging Config to Live Data

**File:** [`fambot/src/memory/injected_models.py`](https://github.com/fambots/famworker/blob/main/fambot/src/memory/injected_models.py)

GraphEngine is the runtime class that brings famgraph.json to life. It's used by both the FastAPI service and Temporal workers.

### Lifecycle

```
1. Construction:
   GraphEngine(uid="user123", src="memory/famgraph.json")

2. Spec loading (in __init__):
   a. Try loading from DB: famgraph_uid table → famgraph table → JSON spec
      (users can have customized famgraphs stored in the DB)
   b. Fallback: load default famgraph.json from disk

3. Graph injection (__inject__):
   a. Parse spec["graph"] into Node/Entity tree
   b. Parse spec["schema"] for column metadata
   c. Resolve filters, sorters, and entity relationships

4. Usage:
   - get_pathways()      → Build Pathway objects for Temporal
   - get_node(path)      → Get a specific node
   - get_path_data()     → Fetch entity data with filters/sorters
   - render_path()       → Render a full view for the API

5. Caching:
   - LRU cache of up to 50 GraphEngine instances (by uid)
   - LRU cache of up to 200 query results
```

### How the API uses GraphEngine

When the frontend requests `GET /fam/root.feed_v2`:

```python
graph = GraphEngine(uid=uid, src="memory/famgraph.json")
return render_path(graph, "root.feed_v2")
```

`render_path` walks the `feed_v2` view node, which references sub-paths like `root.actions`, `root.family_events`, `root.backpack_items`. For each, it runs the SQL query with the node's filters and sorters, and returns the combined result.

---

## 15. The FastAPI Service and Chat

**File:** [`fambot/src/fam/fam_service.py`](https://github.com/fambots/famworker/blob/main/fambot/src/fam/fam_service.py)

### Key Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/fam/{path}` | GET | Fetch data for a graph path (e.g. `root.feed_v2`, `root.actions`) |
| `/fam/{path}` | POST | Create/upsert entity data |
| `/fam/{path}` | PATCH | Update entity data |
| `/fam/{path}` | DELETE | Delete entity data |
| `/chat` | POST | Chat with Fambot (SSE or Socket.IO streaming) |
| `/search/{path}` | GET | Search within a graph path |
| `/last_updated/{path}` | GET | When was this path last updated? |
| `/health` | GET | Health check |
| `/send_digest` | POST | Trigger digest email |

### The Chat System

```
User sends message to POST /chat
  │
  ▼
1. Auth: Verify JWT or service key
2. Budget check: Thread budget ($5) and monthly budget ($40)
3. Load history: From root.chat_threads and root.chat_messages
4. Build system prompt: Include user perception, engagement level, family context
  │
  ▼
5. Invoke "chat" LangGraph agent (Gemini 2.5-pro)
   │
   │  The agent has tools:
   │  ├── ViewFamgraphFromNode — Browse graph data
   │  ├── SearchForEntityItems — Search across entities
   │  ├── ViewEntityItem — View a specific record
   │  ├── InsertEntityItem — Create new records
   │  ├── AddCalendarEvent — Add to Google Calendar
   │  ├── ExploreProfile — View family profiles
   │  ├── AnalyzeImage — Vision analysis
   │  ├── GoogleSearch — Web search
   │  └── ... more tools
   │
   │  The agent loops: think → tool call → observe → think → respond
   ▼
6. Stream response via SSE or Socket.IO
7. Review response quality (Gemini 2.0-flash)
8. Persist perception (how the user seems to be feeling/engaging)
```

### Two Agents

| Agent | Used By | Model | Purpose |
|-------|---------|-------|---------|
| **cleo** | Temporal workers | Varies (per policy) | Data extraction from emails/events |
| **chat** | FastAPI /chat | Gemini 2.5-pro | Interactive chat with users |

Both agents are LangGraph ReAct agents with different tool sets. They share the same codebase but serve different purposes.

---

## 16. Profiles: Family Member Data

**File:** [`fambot/src/memory/profile.py`](https://github.com/fambots/famworker/blob/main/fambot/src/memory/profile.py)

Profiles organize information about family members in a hierarchical structure:

```
root.profiles
  ├── school
  │   ├── identity (name, grade, teacher, school name)
  │   ├── schedule (school hours, pickup/dropoff times)
  │   └── contact_info (teacher email, school phone)
  │
  └── activity
      ├── identity (activity name, team, coach)
      ├── schedule (practice days, game days)
      ├── gear (uniform, equipment)
      └── good_to_know (special instructions)
```

Profile data comes from the LLM extraction pipeline. The `fam_profile` entity stores profile facts with:
- `param_name` — Which profile parameter (e.g. "school.identity.teacher")
- `profile_params` — The structured data
- `cat_ctx` — Category context (which child, which school)
- `attributes` — Additional metadata

Profiles are used for:
- Building rich context for LLM prompts
- Personalizing the chat experience
- Generating the digest

---

## 17. Storage: Postgres, Redis, ClickHouse

### PostgreSQL (primary database)

- ~92 tables defined in famgraph.json schema
- Accessed via SQLAlchemy (`SQLEngine` in [`fambot/src/db/sql/engine.py`](https://github.com/fambots/famworker/blob/main/fambot/src/db/sql/engine.py))
- Supports both sync (psycopg2) and async (asyncpg) access
- Separate read replica for query-heavy workloads

### Redis (caching and data transfer)

- **Data transfer:** Temporal workflows pass large data between PagingWorkflows via Redis keys (data is too large for Temporal's payload limits)
- **Query cache:** `cache_redis` decorator for function-level caching (default TTL: 15 min)
- **GraphEngine:** Uses Redis for query result caching
- Client: `RedisClient` in [`fambot/src/fam/messages/redis_client.py`](https://github.com/fambots/famworker/blob/main/fambot/src/fam/messages/redis_client.py)

### ClickHouse (observability/analytics)

- **`llm_calls`** table — Every LLM invocation: `cost_in_dollars`, `node_path`, `uid`, `model_name`, tokens, duration
- **`gcp_cost_attribution`** table — Daily cost attribution by node_path
- **`gcp_billing_line_items`** table — Raw GCP billing data
- Client: Custom HTTP-based `ClickHouseClient` (not using clickhouse-connect library)
- Used by Grafana dashboards for cost monitoring

---

## 18. Cost Tracking and Observability

### How Costs Are Tracked

Every LLM call goes through `ClickHouseLoggingCallbackHandler`, which:

1. Captures token usage (input, output, cached, reasoning)
2. Looks up the model's price in the `logging_config.pricing` table
3. Computes `cost_in_dollars` from tokens x price
4. Logs to ClickHouse with `node_path` (so you know which pipeline step incurred the cost)

### Dashboard Queries

The Grafana dashboards use queries like:

```sql
-- Cost per node (last 24 hours)
SELECT
    node_path,
    sum(toFloat64(cost_in_dollars)) AS cost
FROM llm_calls
WHERE timestamp >= now() - INTERVAL 24 HOUR
  AND node_path != ''
GROUP BY node_path
ORDER BY cost DESC
```

### Budget Controls (existing)

Currently budgets only exist for chat:
- **Thread budget:** $5.00 per chat thread
- **Monthly budget:** $40.00 per user per month

There is no per-node budget for the pipeline yet (this is what the Node Budget Controller design doc proposes).

---

## 19. Key File Reference

### Workflows and Activities

| File | Purpose |
|------|---------|
| [`src/workflows/chronos.py`](https://github.com/fambots/famworker/blob/main/src/workflows/chronos.py) | Scheduler — picks users for processing |
| [`src/workflows/genesis.py`](https://github.com/fambots/famworker/blob/main/src/workflows/genesis.py) | Per-user orchestrator — runs pathways in phases |
| [`src/workflows/paging.py`](https://github.com/fambots/famworker/blob/main/src/workflows/paging.py) | Data processor — runs one pathway per workflow |
| [`src/workflows/digest_scheduler.py`](https://github.com/fambots/famworker/blob/main/src/workflows/digest_scheduler.py) | Sends daily digest emails |
| [`src/activities/email.py`](https://github.com/fambots/famworker/blob/main/src/activities/email.py) | Fetches emails from Gmail |
| [`src/activities/calendar.py`](https://github.com/fambots/famworker/blob/main/src/activities/calendar.py) | Fetches events from Google Calendar |
| [`src/activities/agent.py`](https://github.com/fambots/famworker/blob/main/src/activities/agent.py) | Runs LLM extraction (agent_activity) |
| [`src/activities/genesis.py`](https://github.com/fambots/famworker/blob/main/src/activities/genesis.py) | Genesis helper activities |
| [`src/activities/partition.py`](https://github.com/fambots/famworker/blob/main/src/activities/partition.py) | Data partitioning for batch processing |
| [`src/activities/billing_sync.py`](https://github.com/fambots/famworker/blob/main/src/activities/billing_sync.py) | Cost attribution to ClickHouse |
| [`src/run_worker.py`](https://github.com/fambots/famworker/blob/main/src/run_worker.py) | Main worker entry — registers workflows and schedules |

### Configuration and Models

| File | Purpose |
|------|---------|
| [`fambot/src/memory/famgraph.json`](https://github.com/fambots/famworker/blob/main/fambot/src/memory/famgraph.json) | Central config: graph nodes, schema, logging |
| [`fambot/src/memory/models.py`](https://github.com/fambots/famworker/blob/main/fambot/src/memory/models.py) | Node, Entity, Graph, EntityNode classes |
| [`fambot/src/memory/policy.py`](https://github.com/fambots/famworker/blob/main/fambot/src/memory/policy.py) | Policy, Pathway, GroundFlow, RecurringTraits |
| [`fambot/src/memory/injected_models.py`](https://github.com/fambots/famworker/blob/main/fambot/src/memory/injected_models.py) | GraphEngine — runtime bridge between config and data |
| [`fambot/src/memory/profile.py`](https://github.com/fambots/famworker/blob/main/fambot/src/memory/profile.py) | Profile resolution and hierarchy |
| [`fambot/src/memory/schema_utils.py`](https://github.com/fambots/famworker/blob/main/fambot/src/memory/schema_utils.py) | Schema inheritance resolution |
| [`fambot/src/memory/composition.py`](https://github.com/fambots/famworker/blob/main/fambot/src/memory/composition.py) | Data composition and merging |

### API and Chat

| File | Purpose |
|------|---------|
| [`fambot/src/fam/fam_service.py`](https://github.com/fambots/famworker/blob/main/fambot/src/fam/fam_service.py) | FastAPI routes (large file — most endpoints) |
| [`fambot/src/graph/chat_graph.py`](https://github.com/fambots/famworker/blob/main/fambot/src/graph/chat_graph.py) | Chat LangGraph agent definition |
| [`fambot/src/graph/chat_system_prompt.py`](https://github.com/fambots/famworker/blob/main/fambot/src/graph/chat_system_prompt.py) | Chat system prompt builder |
| [`fambot/src/fam/fam_tools/chat/`](https://github.com/fambots/famworker/tree/main/fambot/src/fam/fam_tools/chat) | Chat tool implementations |
| [`fambot/src/react_agent/tools.py`](https://github.com/fambots/famworker/blob/main/fambot/src/react_agent/tools.py) | Cleo (Temporal) agent tools |
| [`fambot/src/agents/agents.py`](https://github.com/fambots/famworker/blob/main/fambot/src/agents/agents.py) | Agent registry |

### Database

| File | Purpose |
|------|---------|
| [`fambot/src/db/sql/engine.py`](https://github.com/fambots/famworker/blob/main/fambot/src/db/sql/engine.py) | SQLEngine — database access layer |
| [`fambot/src/db/sql/generated_schema.py`](https://github.com/fambots/famworker/blob/main/fambot/src/db/sql/generated_schema.py) | Auto-generated ORM (do NOT edit) |
| [`fambot/src/db/sql/sync_spec_to_db.py`](https://github.com/fambots/famworker/blob/main/fambot/src/db/sql/sync_spec_to_db.py) | Generates schema from famgraph.json |
| [`fambot/src/db/sql/sync_famgraphs.py`](https://github.com/fambots/famworker/blob/main/fambot/src/db/sql/sync_famgraphs.py) | Syncs famgraph specs to DB |
| [`fambot/src/db/sql/alembic/`](https://github.com/fambots/famworker/tree/main/fambot/src/db/sql/alembic) | Database migrations |

### Observability

| File | Purpose |
|------|---------|
| [`fambot/src/fam/observability/config.py`](https://github.com/fambots/famworker/blob/main/fambot/src/fam/observability/config.py) | Observability configuration |
| [`fambot/src/fam/observability/pricing.py`](https://github.com/fambots/famworker/blob/main/fambot/src/fam/observability/pricing.py) | LLM cost calculation |
| [`fambot/src/fam/observability/llm.py`](https://github.com/fambots/famworker/blob/main/fambot/src/fam/observability/llm.py) | ClickHouse logging callback |
| [`fambot/src/fam/observability/clickhouse_client.py`](https://github.com/fambots/famworker/blob/main/fambot/src/fam/observability/clickhouse_client.py) | ClickHouse HTTP client |
| [`fambot/src/fam/observability/clickhouse/dashboards/`](https://github.com/fambots/famworker/tree/main/fambot/src/fam/observability/clickhouse/dashboards) | Grafana dashboard definitions |

---

## Glossary

| Term | Meaning |
|------|---------|
| **Node** | A location in the famgraph hierarchy that maps to a database table |
| **Entity** | A database table definition (e.g. `email`, `action`, `event`) |
| **Policy** | A processing pipeline attached to a node (source → activity → target) |
| **Pathway** | Runtime representation of a policy, used by PagingWorkflow |
| **GroundFlow** | The core of a policy: flow + source + fields |
| **GraphEngine** | Runtime class that loads famgraph.json and provides data access |
| **Chronos** | Scheduler workflow (picks users) |
| **Genesis** | Per-user orchestrator (runs pathways in phases) |
| **Paging** | Data processor (one per pathway) |
| **Proto entity** | Draft/candidate record (e.g. proto_action → action) |
| **Gist** | Semantic text summary + vector embedding for search/dedup |
| **DDIS** | Daily Digest Interest Score (is there enough new data?) |
| **PCS** | Profile Completeness Score (is the user's profile rich enough?) |
| **Famgraph** | The entire graph configuration system (JSON + runtime) |
| **Feed** | Composed view that aggregates multiple nodes for the frontend |
