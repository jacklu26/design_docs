# Famworker Repository Guide

A step-by-step walkthrough of the famworker codebase — what it does, how data flows, and where everything lives.

---

## Table of Contents

- [Key Concepts at a Glance](#key-concepts-at-a-glance)
1. [What Is Fambot?](#1-what-is-fambot)
2. [Repository Layout](#2-repository-layout)
3. [The Two Main Services](#3-the-two-main-services)
4. [Walkthrough: What Happens When a New User Joins](#4-walkthrough-what-happens-when-a-new-user-joins)
5. [The Workflow Chain: Chronos, Genesis, Paging](#5-the-workflow-chain-chronos-genesis-paging)
6. [The Full Email Pipeline](#6-the-full-email-pipeline)
7. [The Calendar Pipeline](#7-the-calendar-pipeline)
8. [The Daily Digest](#8-the-daily-digest)
9. [Deep Dive: famgraph.json](#9-deep-dive-famgraphjson)
10. [The Graph: Root, Nodes, and Paths](#10-the-graph-root-nodes-and-paths)
11. [Policies and Pathways: How Data Flows Between Nodes](#11-policies-and-pathways-how-data-flows-between-nodes)
12. [The Schema: Database Table Definitions](#12-the-schema-database-table-definitions)
13. [GraphEngine: Bridging Config to Live Data](#13-graphengine-bridging-config-to-live-data)
14. [The FastAPI Service and Chat](#14-the-fastapi-service-and-chat)
15. [Profiles: Family Member Data](#15-profiles-family-member-data)
16. [Storage: Postgres, Redis, ClickHouse](#16-storage-postgres-redis-clickhouse)
17. [Cost Tracking and Observability](#17-cost-tracking-and-observability)
18. [Key File Reference](#18-key-file-reference)

---

## Key Concepts at a Glance

Before diving into the full guide, here are the core abstractions you'll encounter everywhere in the codebase. Understanding these up front will make the rest of the guide (and your code reviews) much easier to follow.

### famgraph.json — The Single Source of Truth

`fambot/src/memory/famgraph.json` (~9,000 lines) is the declarative specification for the entire system. It defines the graph of data nodes, the processing policies that populate them, the database schema for ~92 tables, and the LLM pricing config. Nearly every runtime behavior — what data gets fetched, how it's transformed, where it's stored, and how it's served to the frontend — is driven by this one file. If you only look at one file, make it this one.

### Node — A Data Category in the Graph

A **Node** is a named location in a tree hierarchy, referenced by a dot-separated path like `root.emails` or `root.family.children`. Each node typically maps to a database table (its "entity"), and can carry policies (processing pipelines), filters, sorters, and a row limit. Nodes are the fundamental unit of data organization: the API serves data by node path (`GET /fam/root.feed_v2`), and the processing pipeline targets data into specific nodes.

**Defined in:** `fambot/src/memory/models.py` — `Node` class

### Entity — A Database Table

An **Entity** is the schema definition for a Postgres table (e.g. `email`, `action`, `event`, `subscriber`). Entities are declared in the `schema` section of famgraph.json and inherit columns from base schemas (`system_base`, `gist_base`, `entity_base`, `belief`). The file `sync_spec_to_db.py` auto-generates the SQLAlchemy ORM (`generated_schema.py`) from these definitions — you never edit the ORM by hand.

**Defined in:** `fambot/src/memory/models.py` — `Entity` class; schema in `famgraph.json`

### Policy — A Declarative Processing Rule

A **Policy** is a JSON block attached to a node that declares *how* data flows into it: which source nodes to read from, which activity to run (e.g. `agent_activity`, `calendar_activity`), which fields to extract, which LLM model to use, and the batch size. Policies are the static, config-time description of a pipeline step. Set `"active": false` to disable one without removing it.

**Defined in:** `fambot/src/memory/policy.py` — `Policy` class

### Pathway — The Runtime Pipeline Unit

A **Pathway** is the runtime representation of a policy. When Genesis starts processing a user, `famspec_activity` reads famgraph.json, resolves all active policies, and converts each one into a Pathway dataclass with everything needed for execution: source data key, target node, activity name, LLM model, batch size, field list, filters, and more. Each PagingWorkflow executes exactly one Pathway.

**Defined in:** `fambot/src/memory/policy.py` — `Pathway` class

### GroundFlow — The Core Triple

A **GroundFlow** is the innermost building block of a policy: the combination of a **flow** (which activity to run), a **source** (which node to read from), and the **fields** to extract. A single policy with multiple sources or flows produces multiple GroundFlows, each of which becomes its own Pathway.

**Defined in:** `fambot/src/memory/policy.py` — `GroundFlow` class

### GraphEngine — Config Meets Live Data

**GraphEngine** is the runtime class that loads famgraph.json (or a per-user override from the DB), parses it into a `Graph` of `Node` objects, and provides the methods everyone else calls: `get_pathways()` for Temporal workers, `get_node()` / `render_path()` for the API, and `get_path_data()` for database queries with filters and sorters. It uses LRU caches (50 instances, 200 query results) so it isn't re-parsed on every request.

**Defined in:** `fambot/src/memory/injected_models.py` — `GraphEngine` class

### Chronos, Genesis, Paging — The Workflow Triad

These three Temporal workflows form the heart of the data pipeline:

| Workflow | Role | Cadence |
|----------|------|---------|
| **Chronos** | Scheduler — picks which user to process next | Every 60 seconds |
| **Genesis** | Per-user orchestrator — builds pathways and runs them in phased order | On demand (started by Chronos) |
| **Paging** | Data processor — executes one pathway (fetch, partition, LLM extract, store) | On demand (started by Genesis) |

Chronos decides *who*, Genesis decides *what order*, and Paging does *the actual work*. Data flows between PagingWorkflows via Redis keys (too large for Temporal payloads).

**Defined in:** `src/workflows/chronos.py`, `src/workflows/genesis.py`, `src/workflows/paging.py`

### View Node — Frontend Aggregation

A **View Node** doesn't store its own data. Instead, it references other entity nodes via a `paths` dictionary, each with optional filter and sorter overrides. The primary example is `root.feed_v2`, which pulls data from `root.actions`, `root.family_events`, `root.backpack_items`, `root.insights`, and `root.events` into one combined response for the frontend.

### Proto Entity — Draft Before Final

Many pipelines use a two-pass extraction pattern: the LLM first produces **proto** (draft) records (`root.proto_actions`, `root.proto_events`, etc.), then a second LLM pass refines, deduplicates, and enriches them into final entities (`root.actions`, `root.family_events`). This staged approach improves extraction quality.

### Gist — Semantic Search and Deduplication

A **Gist** is a short text summary of a record plus its vector embedding. Tables that inherit from `gist_base` get `gist`, `gist_embedding`, and `gist_model` columns. Gists power semantic search (finding similar records by meaning) and exclusion-based deduplication (avoiding duplicate extractions across scans).

**Defined in:** `fambot/src/memory/gist_models.py`, inherited via `gist_base` in the schema

### Profiles — Structured Family Data

**Profiles** organize information about family members (children, adults, caregivers) in a hierarchical structure: `school.identity`, `school.schedule`, `activity.gear`, etc. Profile facts are stored as `fam_profile` records and are used to build rich LLM context, personalize chat, compute the Profile Completeness Score (PCS), and generate the daily digest.

**Defined in:** `fambot/src/memory/profile.py`

### DDIS and PCS — Digest Eligibility Scores

Two scores gate whether a user receives a daily digest email. **DDIS** (Daily Digest Interest Score, threshold >= 40) measures whether enough *new data* has arrived since the last digest. **PCS** (Profile Completeness Score, threshold >= 80) measures whether the user's profile is rich enough to produce a personalized, useful digest. Both must be met.

### Activity — A Temporal Unit of Work

An **Activity** is a single function executed by a Temporal worker (e.g. `email_activity` fetches Gmail, `agent_activity` runs an LLM extraction, `partition_activity` splits data into batches). Activities are routed to specific task queues (`activity-task-queue`, `memory-activity-task-queue`, etc.) via the `ActivityMetadataLoader` and task queue router.

**Defined in:** `src/activities/` directory

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

## 4. Walkthrough: What Happens When a New User Joins

Before diving into architecture, let's trace the full journey from signup to seeing data. This makes everything else concrete.

### Step 1: Signup (Firebase)

A user signs up with email and password. Firebase creates an account and sends a verification email.

**Where:** `fambot/src/auth/utils.py` — `create_user_with_email_and_password()`

### Step 2: Onboarding (Subscriber Creation)

The frontend walks the user through onboarding steps: identity, location, coparent, children. Each step POSTs data to `POST /fam/root.subscribers`, which creates a `subscriber` record in Postgres with the user's Firebase `uid`.

**Where:** `fambot/src/fam/fam_service.py` — the `/fam/{path}` POST handler

### Step 3: Connect Google (OAuth)

The user clicks "Connect Google" and goes through the OAuth consent screen. After approval:

1. The callback receives an auth code and exchanges it for tokens
2. A new row is created in the `integration` table: `uid`, `email`, `token`, `active=true`, `last_scan=NULL`
3. The `subscriber` is updated: `connected=true`

**Where:** `fambot/src/fam/fam_service.py` — `/oauth`, `/oauth_ios`, `/oauth/callback` endpoints

The `last_scan=NULL` is the key detail — it's how the system knows this is a brand-new user.

### Step 4: Chronos Picks Up the New User (~60 seconds)

Chronos runs every 60 seconds. Its first action every tick is:

```
"Is there an integration with last_scan=NULL and subscriber.connected=true?"
```

The SQL is:

```sql
SELECT uid FROM integration
WHERE active AND last_scan IS NULL
  AND uid NOT IN (SELECT uid FROM subscriber WHERE NOT connected)
ORDER BY total_scans ASC, last_scan ASC NULLS FIRST
LIMIT 1;
```

New users (first scan) always get priority over rescans, and they skip the concurrency limit (max 2 concurrent Genesis workflows).

**Where:** `src/workflows/chronos.py`, `fambot/src/db/sql/engine.py` — `get_integration_with_greatest_past_due_scan()`

### Step 5: Genesis Processes the User

Chronos starts a GenesisWorkflow for this `uid`. Genesis:

1. Calls `famspec_activity(uid)` to load all pathways from famgraph.json
2. Runs Phase 0 (external): Fetches emails from Gmail and events from Google Calendar
3. Runs Phase 1+ (internal): LLM extracts actions, events, contacts, insights, backpack items
4. All results are stored in Postgres

### Step 6: Data Appears

Once Genesis completes, the user's data is in Postgres. The frontend calls `GET /fam/root.feed_v2` and sees:

- **Actions to Take** — from `root.actions`
- **Key Dates to Add** — from `root.family_events`
- **Good to Know** — from `root.insights`
- **Backpack Check** — from `root.backpack_items`
- **Schedule** — from `root.events`

### Step 7: Ongoing Processing

From now on, Chronos will periodically rescan this user (based on `last_scan` age). The DigestSchedulerWorkflow will start sending daily digest emails once the user's DDIS and PCS scores are high enough.

> **NOTE:** DDIS (Daily Digest Interest Score) measures whether there's enough *new data* since the last digest to make it worthwhile (threshold: >= 40). PCS (Profile Completeness Score) measures whether the user's profile is rich enough — children, schools, activities, etc. — to produce a useful, personalized digest (threshold: >= 80). Both must be met before the first digest is sent.

### Summary Timeline

```
User signup (Firebase)
  ↓ seconds
Onboarding steps (subscriber created)
  ↓ minutes
Connect Google (OAuth → integration with last_scan=NULL)
  ↓ ~60 seconds
Chronos picks up new user
  ↓ immediately
Genesis starts (famspec → pathways → PagingWorkflows)
  ↓ 2-10 minutes
Phase 0: Gmail + Calendar fetched
  ↓
Phase 1+: LLM extraction (emails → actions, events, insights, etc.)
  ↓
Data appears in UI
  ↓ next morning
First daily digest email (if scores are high enough)
```

---

## 5. The Workflow Chain: Chronos, Genesis, Paging

Now that you've seen the user journey, let's look at the three workflows in detail.

### Chronos — The Scheduler

**File:** `src/workflows/chronos.py`

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

This is configured as a Temporal Schedule in `run_worker.py`:

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

**Key concept:** Chronos does NOT process data itself. It only decides WHO gets processed next. The name is a reference to the Greek god of time (it's not an external framework — just a custom Temporal workflow in this repo).

### Genesis — The Per-User Orchestrator

**File:** `src/workflows/genesis.py`

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

### Paging — The Data Processor

**File:** `src/workflows/paging.py`

Paging is where the actual work happens. Each PagingWorkflow handles one pathway (one node + one policy). There are three patterns:

**Pattern A: External Source (Gmail, Calendar)**

```
PagingWorkflow(root.emails):
  1. email_activity() → Call Gmail API, page through emails
  2. Store results in Redis (too large for Temporal)
  3. Signal downstream PagingWorkflows with the Redis key
```

**Pattern B: On-Start (initialization)**

```
PagingWorkflow(on_start):
  1. Send empty batch to targets
  2. Signal Genesis: "I'm done"
```

**Pattern C: Internal Source (node-to-node processing)**

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

## 6. The Full Email Pipeline

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

## 7. The Calendar Pipeline

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

## 8. The Daily Digest

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

## 9. Deep Dive: famgraph.json

**File:** `fambot/src/memory/famgraph.json` (~9000 lines)

This is the single most important file in the repo. Everything else — the workflows, the API, the database — is driven by what's in this file. Think of it as the **declarative specification** for the entire system.

### Top-Level Structure

The file is one big JSON object with three top-level keys:

```json
{
    "logging_config": { ... },   // Lines 2-84      — Observability and pricing
    "graph": { ... },            // Lines 86-3707    — The node tree (data model + pipelines)
    "schema": { ... }            // Lines 3708-8919  — Database table definitions
}
```

### Section 1: `logging_config`

Controls observability and cost tracking:

- **streams** — Which event types to log: `llm_calls`, `activity_executions`, `data_sets`, `function_executions`, `workflow_executions`. Each has an `enabled` flag and `sample_rate`.
- **clickhouse** — Connection timeouts for the analytics database.
- **write** — Buffer settings for log writes: `mode` (async), `buffer_max` (10000), `max_batch` (500), `flush_interval_ms` (250), `drop_policy` ("drop" if buffer is full).
- **pricing** — LLM cost tables per provider/model. For example, Gemini 2.5 Flash costs $0.30 per million input tokens and $2.50 per million output tokens. This is used by the observability layer to calculate `cost_in_dollars` for every LLM call.

### Section 2: `graph`

This defines the node tree — every data category, how they relate to each other, and the policies that drive data processing. This section is what `famspec_activity` reads when Genesis starts. It's covered in detail in the next section.

### Section 3: `schema`

Defines ~92 database tables. Contains:

- **widgets** — UI rendering hints (e.g. `textarea`, `render_google_oauth`)
- **enums** — Enumerated types (e.g. `gender`, `grade`, `caregiver_role`)
- **custom_types** — Complex types like `FieldFilter` (regex-based filtering)
- **bases** — Four base schemas that other entities inherit from (`system_base`, `gist_base`, `entity_base`, `belief`)
- **entities** — The actual table definitions (e.g. `action`, `email`, `event`, `subscriber`, etc.)

This section is covered in detail in section 12 (The Schema).

### How famgraph.json Gets Used

famgraph.json is consumed in several ways:

| Consumer | What It Reads | Purpose |
|----------|---------------|---------|
| `GraphEngine.__init__()` | `graph` + `schema` | Builds the runtime graph, resolves filters/sorters, serves data via API |
| `famspec_activity()` | `graph` (policies) | Builds Pathways for Genesis/Paging workflows |
| `sync_spec_to_db.py` | `schema` | Generates `generated_schema.py` (SQLAlchemy ORM) |
| `PricingTable` | `logging_config.pricing` | Calculates LLM cost per call |
| `ActivityMetadataLoader` | `activity_metadata` | Routes activities to Temporal task queues |
| Tests | `graph` + `schema` | Validates graph topology and entity relationships |

famgraph.json can also be stored **per-user** in the database (in the `famgraph` + `famgraph_uid` tables). This allows different users to have customized graphs. When `GraphEngine` starts, it first checks the DB for a user-specific famgraph; if none is found, it falls back to the default file on disk.

---

## 10. The Graph: Root, Nodes, and Paths

### What Is "root"?

You'll see paths like `root.emails`, `root.family.children`, `root.feed_v2` everywhere. Where does "root" come from?

**"root" is NOT in the JSON file.** The `graph` section of famgraph.json looks like this:

```json
{
    "graph": {
        "profiles": { ... },
        "events": { ... },
        "emails": { ... },
        "family": {
            "children": { "entity": "child" },
            "adults": { "entity": "adult" }
        },
        "actions": { ... },
        "feed_v2": { ... }
    }
}
```

There's no `"root"` key. Instead, "root" is created **at parse time** by the `Node` class. When the `Graph` object is constructed from this JSON, it:

1. Creates a root `Node` with `uid = "root"` (because it has no parent)
2. For each key under `graph` (e.g. `"events"`, `"family"`), creates a child node with `uid = "root." + key`
3. For nested keys (e.g. `"children"` under `"family"`), creates grandchild nodes with `uid = "root.family.children"`

The rule is simple: **path = parent_path + "." + key_name**, starting from "root".

### How Paths Are Built

Here's the code logic (from `fambot/src/memory/models.py`):

```
If no parent exists → node_id = "root"
If parent exists    → node_id = parent_uid + "." + name
```

So for this JSON structure:

```json
"graph": {
    "family": {
        "children": { "entity": "child" },
        "caregivers": { "entity": "caregiver" }
    },
    "emails": { "entity": "email" }
}
```

The parser produces:

```
"root"                  ← the implicit root node
"root.family"           ← from key "family" under root
"root.family.children"  ← from key "children" under "family"
"root.family.caregivers"← from key "caregivers" under "family"
"root.emails"           ← from key "emails" under root
```

All nodes are registered in `graph.all_nodes`, a flat dictionary mapping path strings to `Node` objects. This is how `GraphEngine.get_node("root.family.children")` works — it's a direct lookup.

### Anatomy of a Node

A **node** is a location in the graph hierarchy. Here's what a node can contain:

```json
"events": {
    "entity": "event",
    "description": "Events from calendar accounts.",
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
    ],
    "filters": [...],
    "sorters": [...],
    "limit": 100
}
```

| Field | Purpose |
|-------|---------|
| `entity` | The database table this node maps to (e.g. `"event"` → the `event` table) |
| `description` | Human-readable description of what this node stores |
| `policies` | Processing pipelines that feed data into this node (see section 11) |
| `filters` | Default WHERE clause filters applied when querying this node |
| `sorters` | Default ORDER BY when querying this node |
| `limit` | Max rows to return |
| `paths` | (View nodes only) References to other nodes, with optional filter overrides |
| `external_entity` | (External nodes only) E.g. `"google_calendar"` — queries a live API |

### Types of Nodes

**Entity nodes** — The most common type. They have an `entity` (database table) and optionally `policies` (processing pipelines). Examples: `root.emails`, `root.actions`, `root.events`.

**View nodes** — Aggregation nodes for the frontend. They don't store data themselves but reference other nodes via a `paths` dictionary. For example, `root.feed_v2` is a view that pulls in data from `root.actions`, `root.family_events`, `root.backpack_items`, `root.insights`, `root.events`, etc. — each with its own filters and sorters:

```json
"feed_v2": {
    "paths": {
        "root.actions": {
            "filters": [{"final": true}, {"deadline": {">=": "CURRENT_DATE"}}]
        },
        "root.backpack_items": {
            "filters": [{"final": true}]
        },
        "root.events": {
            "filters": [...]
        }
    }
}
```

When the API serves `GET /fam/root.feed_v2`, it deep-copies each referenced node, applies the filter overrides, queries Postgres for each, and returns the combined result.

**Profile nodes** — Under `root.profiles`, these define schemas for family member information (school identity, schedules, gear, etc.).

**External nodes** — Like `root.google_events`, which has `"external_entity": "google_calendar"`. These query a live API instead of Postgres.

### Node Catalog (major nodes)

```
Ingestion:
  root.emails                → Raw emails from Gmail
  root.events                → Raw calendar events from Google Calendar
  root.google_events         → Live Google Calendar API (external)

Processing (LLM extraction):
  root.family_emails         → Family-relevant emails (filtered by LLM)
  root.email_summaries       → Newsletter summaries
  root.proto_events          → Candidate events extracted from emails
  root.proto_actions         → Candidate action items from emails
  root.proto_contacts        → Candidate contacts from emails
  root.proto_insights        → Candidate "good to know" items
  root.proto_backpack_items  → Candidate backpack checklist items

Final entities (user-facing):
  root.family_events         → Key Dates to Add
  root.actions               → Actions to Take / On the Horizon
  root.contacts              → Key Contacts
  root.insights              → Good to Know
  root.backpack_items        → Daily Backpack Check

Family data:
  root.family.children       → Children in the family
  root.family.adults         → Adults
  root.family.caregivers     → Caregivers
  root.family.pets           → Pets
  root.family_activities     → Ongoing activities (sports, clubs)
  root.actor_roles           → Who does what in each activity
  root.fam_profiles          → Profile facts

System:
  root.subscribers           → Users
  root.accounts.google       → Google OAuth integrations
  root.calendars             → Calendar preferences
  root.notification_settings → Notification preferences
  root.chat_threads          → Chat conversations
  root.chat_messages         → Chat messages
  root.weather               → Weather data
  root.digest_teasers        → Pre-computed digest content

Views (frontend aggregations):
  root.feed_v2               → Main feed (actions + events + insights + backpack + schedule)
  root.newsletters           → Newsletter decoder view
```

---

## 11. Policies and Pathways: How Data Flows Between Nodes

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

## 12. The Schema: Database Table Definitions

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

## 13. GraphEngine: Bridging Config to Live Data

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

## 14. The FastAPI Service and Chat

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

## 15. Profiles: Family Member Data

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

## 16. Storage: Postgres, Redis, ClickHouse

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

## 17. Cost Tracking and Observability

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

## 18. Key File Reference

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
