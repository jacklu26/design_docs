# How Famworker Actually Works: A Case Study

Follow one user's journey from signup to seeing "Actions to Take" in their feed. Along the way, we'll meet every major abstraction in the system — not as definitions, but as the things that make this journey happen.

---

## Part 1: The Story — Sarah Signs Up

Sarah is a parent with two kids. She signs up for Fambot, goes through onboarding, and connects her Gmail account. At this point, a row is created in the `integration` table:

```
uid: "sarah_123"    email: "sarah@gmail.com"    active: true    last_scan: NULL
```

That `last_scan: NULL` is the trigger for everything that follows. Sarah closes her browser and goes about her day. She has no idea what happens next — but within 10 minutes, her feed will show extracted action items from her school newsletters.

Let's follow that journey.

---

## Part 2: How It Gets Triggered — The Production Mechanism

Nothing in this system runs because of a user click. There is no "process my emails" button. Instead, a **scheduled loop** runs constantly in the background.

### Chronos: The 60-Second Heartbeat

A Temporal workflow called **Chronos** runs every 60 seconds. Every tick, it asks one question:

> "Is there a user who needs processing?"

The SQL looks roughly like:

```sql
SELECT uid FROM integration
WHERE active AND last_scan IS NULL
  AND uid NOT IN (SELECT uid FROM subscriber WHERE NOT connected)
ORDER BY total_scans ASC, last_scan ASC NULLS FIRST
LIMIT 1;
```

New users (`last_scan IS NULL`) always get priority. Sarah's row matches immediately on the next tick — within 60 seconds of connecting her Gmail.

Chronos doesn't process Sarah's data itself. It starts a **Genesis** workflow for her:

```python
await workflow.execute_child_workflow(
    GenesisWorkflow.run,
    GenesisRequest(uid="sarah_123", ...),
    ...
)
```

**File:** `src/workflows/chronos.py`

### Genesis: Sarah's Personal Orchestrator

Genesis is a per-user workflow. Its job: figure out *what* processing needs to happen for Sarah, and *in what order*.

But how does Genesis know what to do? It doesn't have hard-coded logic like "first fetch emails, then extract actions." Instead, it asks a configuration file.

```python
@workflow.run
async def run(self, request: GenesisRequest) -> None:
    flowgraph_params = await workflow.execute_activity(
        famspec_activity,
        FamspecParams(uid=request.uid),
        ...
    )
    for phase_idx, phase_pathways in enumerate(flowgraph_params.pathways_by_phase):
        await self._run_phase(request, phase_pathways)
```

That `famspec_activity` call is where the configuration system enters the picture.

**File:** `src/workflows/genesis.py`

---

## Part 3: The Configuration — Where famgraph.json Comes In

### What famspec_activity actually does

```python
@activity.defn()
async def famspec_activity(famspec: FamspecParams) -> FlowgraphParams:
    graph = await GraphEngine.create_async(famspec.uid)
    pathways_by_phase = await graph.get_pathways_by_phases()
    return FlowgraphParams(famspec.uid, pathways_by_phase)
```

It loads a file called `famgraph.json` — a ~9,000-line JSON file that is the **single source of truth** for the entire system. This file declares every data category, every processing pipeline, every database table, and how they all connect.

**File:** `fambot/src/memory/famgraph.json`

### Nodes: The data categories

Inside `famgraph.json`, there's a `graph` section that defines **nodes** — named data categories organized in a tree. Each node has a dot-separated path:

```
root.emails              → Raw emails from Gmail
root.family_emails       → Family-relevant emails (filtered by LLM)
root.proto_actions       → Draft action items (extracted by LLM)
root.actions             → Final action items (refined by LLM)
root.events              → Calendar events from Google Calendar
root.family_events       → Key dates extracted from emails
root.insights            → "Good to Know" items
root.backpack_items      → Daily backpack checklist
```

Each node maps to a Postgres table via its `entity` field. For example, `root.actions` has `"entity": "action"`, meaning it reads from and writes to the `action` table.

### Policies: How data flows between nodes

Nodes don't just store data — they declare *how* they get populated. Each node can have one or more **policies** that say: "To fill me, take data from *this* source, run *this* activity, and extract *these* fields."

Here's the real policy on the `actions` node in famgraph.json:

```json
"actions": {
    "entity": "action",
    "description": "Pick out the real-world action that matters most...",
    "policies": [
        {
            "name": "Action Policy",
            "active": true,
            "batch_size": 1,
            "data_query": "root.proto_actions",
            "merge_by_gist": true,
            "traits": {
                "flows": ["agent_activity"],
                "sources": ["root.proto_actions"],
                "fields": ["name", "description", "cta", "start_time",
                           "location", "startline", "deadline"]
            }
        }
    ]
}
```

Reading this declaratively: "To populate `root.actions`, read from `root.proto_actions`, run `agent_activity`, and extract these 7 fields." No Python code tells the system to do this — it's all declared in JSON.

Every arrow in the processing chain is a policy:

```
root.emails                    root.family_emails              root.proto_actions           root.actions
  policy: email_activity  →      policy: agent_activity   →     policy: agent_activity  →    policy: agent_activity
  source: external               source: root.emails            source: root.family_emails   source: root.proto_actions
  model: n/a (API call)          model: gemini-2.5-flash        model: gemini-2.5-flash      model: (default)
  batch_size: n/a                batch_size: 4                  batch_size: 1                batch_size: 1
```

### From Policy to Pathway

Policies are the static, config-time declaration. But Temporal workflows need a runtime object they can pass around, serialize, and act on. That runtime object is called a **Pathway**.

When `famspec_activity` calls `graph.get_pathways_by_phases()`, here's what happens internally:

1. **Traits resolve into GroundFlows.** Each policy has `traits` (flows, sources, fields). These get combined into `GroundFlow` objects — the core triple of *which activity* + *which source* + *which fields*:

```python
class GroundFlow(BaseModel):
    flow: Flow           # e.g. agent_activity
    source: NodeQuery    # e.g. root.proto_actions
    fields: list[str]    # e.g. [name, description, cta, ...]
```

2. **Each GroundFlow becomes a Pathway.** The function `ground_flow_to_pathway()` takes the GroundFlow plus all the policy-level config (batch_size, model, derived_fields, merge_by_gist, etc.) and creates a `Pathway` — a flat dataclass with ~80 fields carrying *everything* needed for execution.

3. **Pathways are grouped by phase.** Phase 0 = external sources (Gmail, Calendar). Phase 1+ = internal processing, ordered by data dependencies.

For Sarah's scan, `famspec_activity` returns something like:

```
Phase 0 (parallel):
  - Pathway: root.emails       (source: external, flow: email_activity)
  - Pathway: root.events       (source: external, flow: calendar_activity)

Phase 1 (sequential):
  - Pathway: root.family_emails      (source: root.emails, flow: agent_activity)
  - Pathway: root.proto_actions      (source: root.family_emails, flow: agent_activity)
  - Pathway: root.actions            (source: root.proto_actions, flow: agent_activity)
  - Pathway: root.proto_events       (source: root.email_summaries, flow: agent_activity)
  - Pathway: root.family_events      (source: root.proto_events, flow: agent_activity)
  - ... ~15 more pathways
```

**Files:** `fambot/src/memory/policy.py` (Policy, GroundFlow, Pathway), `src/activities/famspec.py`

---

## Part 4: Execution — PagingWorkflow Does the Work

Genesis now has a list of Pathways grouped by phase. It starts one **PagingWorkflow** per Pathway.

### How Genesis orchestrates

```python
async def _run_phase(self, request, phase_pathways):
    for pathway in phase_pathways:
        if pathway.source in ["external", "on_start"]:
            # Start in parallel — don't wait
            external_workflows.append(
                workflow.start_child_workflow(PagingWorkflow.run, ...)
            )
        else:
            # Start sequentially — wait for "ready" signal before starting the next
            child = workflow.start_child_workflow(PagingWorkflow.run, ...)
            await child
            await workflow.wait_condition(lambda: self._child_workflows_ready[wf_id])

    await asyncio.gather(*external_workflows)
    await workflow.wait_condition(lambda: all_complete())
```

Phase 0 runs Gmail and Calendar fetches in parallel. Phase 1+ runs internal pathways one at a time, because each depends on the previous one's output.

### What PagingWorkflow does (for root.actions)

Each PagingWorkflow receives exactly one Pathway. For `root.actions`:

**1. Signal "ready" to Genesis** — so Genesis can start the next PagingWorkflow in sequence.

**2. Wait for upstream data** — the `root.proto_actions` PagingWorkflow sends a signal with a Redis key pointing to the proto_action records it produced.

**3. Partition** — `partition_activity` splits the data by `partition_keys` (here: `["uid"]`) into batches sized according to `batch_size`.

**4. Call the agent** — this is the LLM call. PagingWorkflow dynamically invokes whatever activity is named in the Pathway:

```python
self.signal_activity = request.pathway.workflow.name  # "agent_activity"
# ...
response = await workflow.execute_activity(
    self.signal_activity,  # dynamically resolved from famgraph.json
    request,
    task_queue=self.pathway.signal_activity_task_queue,
    ...
)
```

The activity name isn't hard-coded — it comes from the policy's `traits.flows` in famgraph.json. This is how the JSON config declaratively controls runtime behavior.

**5. agent_activity runs the LLM:**

```python
@activity.defn
async def agent_activity(request: TargetRequest) -> AgentActivityResponse:
    graph = await GraphEngine.create_async(request.pathway.uid)
    response = await run_agent_activity(request)
    return response
```

Inside `run_agent_activity`:
- Builds a prompt from the pathway's `description`, `fields`, source data, and family context
- Calls the LLM (model specified in the pathway, e.g. gemini-2.5-flash)
- Uses tool-calling (`InsertNodeEntity`) to get structured output
- Returns extracted records matching the 7 fields: `name`, `description`, `cta`, `start_time`, `location`, `startline`, `deadline`

**6. Write to Postgres** — the extracted action records are written to the `action` table with metadata like `node_path="root.actions"`, `policy_name="Action Policy"`, and `scan_sequence_number`.

**7. Deduplication** — because `merge_by_gist: true`, the system computes a gist embedding for each record and checks for similar existing records before inserting. This prevents "Sign permission slip by Friday" from appearing twice across scans.

**8. Signal Genesis "complete":**

```python
await self._signal_with_retry(
    self.genesis_wf_id, "workflow_complete",
    WorkflowCompleteRequest(workflow_id=wf_id, status="Completed")
)
```

Genesis marks this PagingWorkflow as done and, once all pathways in the phase complete, moves to the next phase.

**Files:** `src/workflows/paging.py`, `src/activities/agent.py`, `src/activities/partition.py`

---

## Part 5: GraphEngine — The Runtime Bridge

You've seen `GraphEngine` appear several times:

- In `famspec_activity`: `graph = await GraphEngine.create_async(uid)` → builds Pathways
- In `agent_activity`: `graph = await GraphEngine.create_async(uid)` → provides context
- In the API: `graph = GraphEngine(uid=uid)` → serves data to the frontend

GraphEngine is the **runtime class that brings famgraph.json to life**. It:

1. **Loads the spec** — tries the DB first (users can have customized graphs), falls back to the JSON file on disk
2. **Parses the graph** — builds `Node` and `Entity` objects from the `graph` section
3. **Parses the schema** — resolves table definitions and base inheritance from the `schema` section
4. **Exposes methods** for different consumers:

| Method | Used by | Purpose |
|--------|---------|---------|
| `get_pathways()` | Temporal workers | Build Pathway objects from all active policies |
| `get_pathways_by_phases()` | Genesis via famspec_activity | Pathways grouped by execution phase |
| `get_node(path)` | API, workers | Look up a specific node by path |
| `get_path_data(path)` | API | Query Postgres with the node's filters and sorters |
| `render_path(path)` | API | Render a full view (resolving view nodes into combined data) |

GraphEngine uses LRU caches (50 instances by uid, 200 query results) so the 9,000-line JSON isn't re-parsed on every request.

**File:** `fambot/src/memory/injected_models.py`

---

## Part 6: Sarah Sees Her Data

After Genesis completes all phases (~2–10 minutes), Sarah's data is in Postgres. When she opens the app, the frontend calls:

```
GET /fam/root.feed_v2
```

The FastAPI service creates a GraphEngine for Sarah's uid, looks up the `root.feed_v2` node, and discovers it's a **view node** — it doesn't store data itself but references other nodes:

```json
"feed_v2": {
    "paths": {
        "root.actions":        { "filters": [{"final": true}, {"deadline": {">=": "CURRENT_DATE"}}] },
        "root.family_events":  { "filters": [{"final": true}] },
        "root.backpack_items": { "filters": [{"final": true}] },
        "root.insights":       { "filters": [{"final": true}] },
        "root.events":         { "filters": [...] }
    }
}
```

GraphEngine queries each referenced node from Postgres (with the filter overrides), and returns the combined result. Sarah sees:

- **Actions to Take** — "Sign permission slip by Friday", "Pay field trip fee by March 28"
- **Key Dates** — "Spring Concert, April 15 at 6pm"
- **Good to Know** — "New dismissal procedure starts Monday"
- **Backpack Check** — "Soccer cleats, water bottle, signed form"

All of this was extracted from her Gmail by the pipeline we just traced.

---

## Putting It All Together

Here's the complete flow, from signup to screen:

```
Sarah connects Gmail
  │
  ▼  (integration row: last_scan=NULL)
Chronos (every 60s) picks Sarah
  │
  ▼
Genesis starts for sarah_123
  │
  ├─ famspec_activity loads famgraph.json via GraphEngine
  │    └─ Parses nodes, policies → GroundFlows → Pathways
  │    └─ Groups ~20 Pathways into phases
  │
  ├─ Phase 0 (parallel):
  │    ├─ PagingWorkflow(root.emails)  → Gmail API → raw emails to Postgres
  │    └─ PagingWorkflow(root.events)  → Calendar API → events to Postgres
  │
  ├─ Phase 1 (sequential, each waits for upstream):
  │    ├─ PagingWorkflow(root.family_emails)    → LLM filters for family emails
  │    ├─ PagingWorkflow(root.proto_actions)    → LLM extracts draft actions
  │    ├─ PagingWorkflow(root.actions)          → LLM refines into final actions
  │    ├─ PagingWorkflow(root.proto_events)     → LLM extracts draft events
  │    ├─ PagingWorkflow(root.family_events)    → LLM refines into final events
  │    └─ ... more pathways for insights, contacts, backpack items
  │
  └─ All phases complete → last_scan updated
       │
       ▼
Sarah opens the app
  │
  ▼  GET /fam/root.feed_v2
GraphEngine resolves the view node
  │
  ├─ Queries root.actions      → "Sign permission slip by Friday"
  ├─ Queries root.family_events → "Spring Concert, April 15"
  ├─ Queries root.insights      → "New dismissal procedure"
  └─ Queries root.backpack_items → "Soccer cleats, water bottle"
       │
       ▼
Sarah sees her personalized family feed
```

---

## Concept Index

Every abstraction we met, in the order it appeared:

| Concept | What it is | Where we met it |
|---------|-----------|-----------------|
| **Chronos** | Temporal workflow that runs every 60s, picks the next user to process | Part 2 |
| **Genesis** | Per-user Temporal workflow that orchestrates all processing | Part 2 |
| **famgraph.json** | ~9,000-line JSON file declaring all nodes, policies, and schema | Part 3 |
| **Node** | Named data category in a tree (e.g. `root.actions` → `action` table) | Part 3 |
| **Policy** | Declarative rule on a node: source → activity → fields | Part 3 |
| **GroundFlow** | Core triple: which activity + which source + which fields | Part 3 |
| **Pathway** | Runtime dataclass (~80 fields) carrying everything for execution | Part 3 |
| **PagingWorkflow** | Temporal workflow that executes one Pathway | Part 4 |
| **agent_activity** | Temporal activity that calls the LLM for extraction | Part 4 |
| **partition_activity** | Temporal activity that splits data into batches | Part 4 |
| **Gist / merge_by_gist** | Semantic embedding for deduplication across scans | Part 4 |
| **GraphEngine** | Runtime class that loads famgraph.json and serves data | Part 5 |
| **View Node** | Aggregation node (like `root.feed_v2`) that combines other nodes | Part 6 |

---

## Key Files

| File | What it does |
|------|-------------|
| `fambot/src/memory/famgraph.json` | The source of truth — nodes, policies, schema |
| `fambot/src/memory/policy.py` | Policy, GroundFlow, Pathway classes |
| `fambot/src/memory/injected_models.py` | GraphEngine — loads config, builds pathways, serves data |
| `fambot/src/memory/models.py` | Node, Entity, Graph classes |
| `src/workflows/chronos.py` | Scheduler — picks users every 60s |
| `src/workflows/genesis.py` | Per-user orchestrator — runs pathways in phases |
| `src/workflows/paging.py` | Data processor — one per pathway |
| `src/activities/famspec.py` | Loads GraphEngine, converts policies to pathways |
| `src/activities/agent.py` | Runs the LLM extraction |
| `src/activities/partition.py` | Splits data into batches |
| `fambot/src/fam/fam_service.py` | FastAPI — serves `GET /fam/{path}` to the frontend |
