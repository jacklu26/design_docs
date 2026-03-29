# Pathway End-to-End: From famgraph.json to Postgres

A hands-on walkthrough tracing a single Pathway through the entire system, using `root.actions` as the concrete example. Read the [Famworker Guide](famworker_guide.md) first for foundational concepts.

---

## The Chain We're Tracing

```
root.emails → root.family_emails → root.proto_actions → root.actions
                                                            ↑
                                                     (this is our target)
```

Each arrow is a separate Policy → Pathway → PagingWorkflow → LLM call. We'll follow `root.actions` (the final link), but the same mechanics apply to every link in the chain.

---

## Step 1: The Policy in famgraph.json

Everything starts in `fambot/src/memory/famgraph.json`. The `actions` node (which becomes `root.actions` at runtime) is defined around line 885:

```json
"actions": {
    "entity": "action",
    "description": "Pick out the real-world action that matters most from the events...",
    "gist_prompt": "Describe what needs to happen in 4-8 words. Keep it simple.",
    "policies": [
        {
            "name": "Action Policy",
            "active": true,
            "batch_size": 1,
            "data_query": "root.proto_actions",
            "merge_by_gist": true,
            "derived_fields": [
                { "derived_date": "inheritOrParse(start_time)" },
                { "source_cb_map": "inherit(source_cb_map)" },
                { "deadline": "fallback_chain(start_time, date_add(startline, 1, day))" }
            ],
            "traits": {
                "flows": ["agent_activity"],
                "sources": ["root.proto_actions"],
                "fields": ["name", "description", "cta", "start_time", "location", "startline", "deadline"]
            }
        }
    ]
}
```

### What each field means

| Field | Value | Purpose |
|-------|-------|---------|
| `entity` | `"action"` | Writes to the `action` Postgres table |
| `traits.sources` | `["root.proto_actions"]` | Input comes from upstream proto_actions |
| `traits.flows` | `["agent_activity"]` | Runs the LLM extraction activity |
| `traits.fields` | 7 fields | What the LLM must extract from each input record |
| `batch_size` | `1` | Process one proto_action at a time |
| `merge_by_gist` | `true` | Deduplicate via gist embeddings across scans |
| `data_query` | `"root.proto_actions"` | Where to query source data from Postgres |
| `derived_fields` | fallback chains | Auto-computed fields (e.g. deadline falls back to start_time) |

### The upstream chain

The `root.actions` node doesn't exist in isolation. Here's how data flows to it:

**root.family_emails** (line ~2229) — filters raw emails for family-relevant content:
- `model: "google_genai/gemini-2.5-flash"`, `batch_size: 4`
- Source: `root.emails`, extracts: `name, summary, salience, follow_links`

**root.proto_actions** (line ~2046) — extracts draft action items from family emails:
- `model: "google_genai/gemini-2.5-flash"`, `batch_size: 1`
- Source: `root.family_emails`, extracts: `name`
- Detailed description instructs the LLM on what counts as an action vs. noise

**root.actions** (line ~885) — refines proto_actions into final, user-facing actions:
- No explicit model (uses default), `batch_size: 1`
- Source: `root.proto_actions`, extracts all 7 fields
- `merge_by_gist: true` for cross-scan deduplication

---

## Step 2: Policy Becomes a Pathway

When Chronos triggers Genesis for a user, the first thing Genesis does is call `famspec_activity` (`src/activities/famspec.py`):

```python
@activity.defn()
async def famspec_activity(famspec: FamspecParams) -> FlowgraphParams:
    graph = await GraphEngine.create_async(famspec.uid)
    await graph.engine.set_integrations_scan_to_now(famspec.uid)
    pathways_by_phase = await graph.get_pathways_by_phases()
    # ... optional target filtering ...
    res = FlowgraphParams(famspec.uid, pathways_by_phase)
    return res
```

### The conversion chain: traits → GroundFlow → Pathway

Inside `get_pathways_by_phases()` (in `injected_models.py`), for each node:

**1. Policy validation resolves traits into GroundFlows.** When the `Policy` Pydantic model is constructed, it auto-resolves `traits` into `GroundFlow` objects via `resolve_ground_flows()`. Each GroundFlow is the core triple:

```python
class GroundFlow(BaseModel):
    flow: Flow           # which activity to run (e.g. agent_activity)
    source: NodeQuery    # which node to read from (e.g. root.proto_actions)
    fields: list[str]    # what fields to extract
    prompt: Prompt       # prompt template (from node description)
    targets: list[str]   # downstream nodes to signal
```

For our Action Policy, there's one GroundFlow: `flow=agent_activity`, `source=root.proto_actions`, `fields=[name, description, cta, ...]`.

**2. Each GroundFlow becomes a Pathway.** The function `ground_flow_to_pathway()` (policy.py, line ~1605) takes the GroundFlow plus all policy-level config and creates a `Pathway` — a flat dataclass with ~80 fields carrying everything PagingWorkflow needs:

```python
def ground_flow_to_pathway(id, name, gf, batch_size, model, ...) -> Pathway:
    return Pathway(
        id=id,
        name=name,
        workflow=flow_to_workflow(gf.flow),   # maps "agent_activity" → workflow config
        fields=gf.fields,
        source=gf.source.path,
        node_path=node_path,                  # "root.actions"
        data_query=data_query,                # "root.proto_actions"
        model=model,
        batch_size=batch_size,
        merge_by_gist=merge_by_gist,
        # ... ~70 more fields ...
    )
```

**3. Pathways are grouped by phase.** `get_pathways_by_phases()` buckets pathways by their `phase` field. Phase 0 is external sources (Gmail, Calendar), Phase 1+ is internal processing in dependency order.

---

## Step 3: Genesis Orchestrates the Phases

Genesis (`src/workflows/genesis.py`) processes phases sequentially:

```python
@workflow.run
async def run(self, request: GenesisRequest) -> None:
    flowgraph_params = await workflow.execute_activity(famspec_activity, ...)

    for phase_idx, phase_pathways in enumerate(flowgraph_params.pathways_by_phase):
        selected_pathways = select_phase_pathways(phase_pathways, genesis_failed)
        await self._run_phase(request, selected_pathways)
```

In `_run_phase`, the key distinction:

- **External pathways** (source = `"external"` or `"on_start"`) — started **in parallel** via `asyncio.gather`
- **Internal pathways** (like `root.actions`) — started **sequentially**, each one signals "ready" before the next starts

```python
if pathway.source == "external" or pathway.source == "on_start":
    child_workflow = workflow.start_child_workflow(PagingWorkflow.run, ...)
    external_workflows.append(child_workflow)
else:
    child_workflow = workflow.start_child_workflow(PagingWorkflow.run, ...)
    await child_workflow
    # Wait for this workflow to signal it's ready before starting the next
    await workflow.wait_condition(
        lambda wf=wf_id: self._child_workflows_ready[wf],
        timeout=timedelta(seconds=3600),
    )
```

Genesis tracks each child workflow in `_child_workflows_complete` and waits for all to signal `workflow_complete` before moving to the next phase.

---

## Step 4: PagingWorkflow Executes the Pathway

Each PagingWorkflow (`src/workflows/paging.py`) receives exactly one Pathway. For internal sources like `root.actions`:

### a) Signal Genesis "ready" and send empty init batch

```python
if request.pathway.source not in ["external", "on_start"]:
    self.signal_activity = request.pathway.workflow.name  # "agent_activity"
    if self.genesis_wf_id is not None:
        await self._signal_with_retry(self.genesis_wf_id, "child_ready", ...)
    await self.signal_response_to_targets({}, init_request, send_empty=True)
```

### b) Wait for upstream data

PagingWorkflow waits for signals from `root.proto_actions`'s PagingWorkflow, which passes Redis keys pointing to the source data.

### c) Partition the data

`partition_activity` splits incoming data by `partition_keys` (here: `["uid"]`):

```python
partitioned_data = await workflow.execute_activity(
    partition_activity, partition_params,
    task_queue="activity-task-queue", ...
)
```

### d) Call the agent (LLM extraction)

For each partition, `call_activity` → `exec_signal_activity` dynamically invokes the activity named in the pathway:

```python
async def exec_signal_activity(self, request, ...):
    response = await workflow.execute_activity(
        self.signal_activity,  # "agent_activity" (from pathway.workflow.name)
        request,
        task_queue=self.pathway.signal_activity_task_queue,
        start_to_close_timeout=timedelta(seconds=720),
        ...
    )
```

Note: `self.signal_activity` is a **string** — the activity name from the pathway config. This is how famgraph.json declaratively controls which activity runs for each node.

### e) Signal Genesis "complete"

```python
async def signal_workflow_end_to_genesis(self):
    wf_completion_request = WorkflowCompleteRequest(
        workflow_id=wf_id, status="Completed"
    )
    await self._signal_with_retry(
        self.genesis_wf_id, "workflow_complete", wf_completion_request
    )
```

---

## Step 5: agent_activity — The LLM Call

`agent_activity` (`src/activities/agent.py`) is the Temporal activity that runs the LLM:

```python
@activity.defn
async def agent_activity(request: TargetRequest) -> AgentActivityResponse:
    graph = await GraphEngine.create_async(request.pathway.uid)
    ctx_settings = await graph.get_contextual_settings()
    request.pathway.activity_instance_id = uuid.uuid4().hex
    response = await run_agent_activity(request)
    # Cleanup heavy fields to save Temporal payload size
    response.pathway.category_contexts = None
    response.pathway.source_content = None
    response.pathway.prompt = og_prompt
    return response
```

The heavy lifting is in `run_agent_activity` (from `fam.fam_tools.agent_tool_utils`), which:

1. **Builds a prompt** from the pathway's `description`, `fields`, `gist_prompt`, and source data
2. **Calls the LLM** using the model from the pathway (or default)
3. **Uses tool-calling** (typically the `InsertNodeEntity` tool) to get structured output matching the field list
4. **Returns extracted records** as an `AgentActivityResponse`

For `root.actions`, the LLM receives proto_action records and must extract: `name`, `description`, `cta`, `start_time`, `location`, `startline`, `deadline`.

---

## Step 6: Results Land in Postgres

Extracted records are written to the `action` table (the `entity` from the node definition). Every record inherits columns from `system_base`:

| Column | Value | Source |
|--------|-------|--------|
| `node_path` | `"root.actions"` | From the pathway |
| `policy_name` | `"Action Policy"` | From the pathway |
| `pathway_id` | UUID | From the pathway |
| `scan_sequence_number` | Integer | Which scan cycle created this |
| `final` | Boolean | Whether this record is finalized |
| `git_commit` | String | Code version that created this |
| `created_at` / `updated_at` | Timestamps | Auto-set |

Because `merge_by_gist: true`, the system also:
1. Computes a gist embedding for the new record
2. Searches for existing records with similar gists (above `gist_threshold: 0.80`)
3. If a match is found, merges instead of inserting a duplicate

### derived_fields resolution

Before writing, derived fields are resolved. For example:
- `"deadline": "fallback_chain(start_time, date_add(startline, 1, day))"` — if the LLM didn't extract a deadline, compute it from start_time, or from startline + 1 day
- `"source_cb_map": "inherit(source_cb_map)"` — carry forward the source callback map from the upstream proto_action

---

## The Complete Picture

```
famgraph.json
  └─ "actions" node with "Action Policy"
       │
       ▼  famspec_activity (Temporal activity)
  Policy.traits → GroundFlow → ground_flow_to_pathway() → Pathway
       │
       ▼  Genesis groups by phase, starts child workflow
  PagingWorkflow receives Pathway
       │
       ├─ signals Genesis "ready"
       ├─ waits for upstream signal (from root.proto_actions PagingWorkflow)
       ├─ partition_activity() splits data by uid
       ├─ agent_activity() → run_agent_activity() → LLM call
       │    └─ extracts: name, description, cta, start_time, location, startline, deadline
       │    └─ uses InsertNodeEntity tool for structured output
       ├─ derived_fields resolved (deadline fallback, source_cb_map inheritance)
       ├─ merge_by_gist deduplication check
       ├─ results written to `action` table in Postgres
       ├─ signals downstream targets (if any)
       └─ signals Genesis "workflow_complete"
```

---

## Key Files Referenced

| File | Role in this flow |
|------|-------------------|
| `fambot/src/memory/famgraph.json` | Source of truth — node + policy definitions |
| `fambot/src/memory/policy.py` | Policy, GroundFlow, Pathway classes; `ground_flow_to_pathway()` |
| `fambot/src/memory/injected_models.py` | GraphEngine; `get_pathways()`, `get_pathways_by_phases()` |
| `src/activities/famspec.py` | `famspec_activity` — loads graph, builds pathways |
| `src/workflows/genesis.py` | GenesisWorkflow — orchestrates phases, starts PagingWorkflows |
| `src/workflows/paging.py` | PagingWorkflow — partitions, calls agent, signals completion |
| `src/activities/agent.py` | `agent_activity` — runs the LLM extraction |
| `src/activities/partition.py` | `partition_activity` — splits data into batches |

---

## What to Explore Next

- **How prompts are built:** Trace `run_agent_activity` in `fambot/src/fam/fam_tools/agent_tool_utils.py` to see how the node description, fields, and source data become an LLM prompt.
- **Gist deduplication:** Look at `merge_by_gist` handling in the agent tool chain to understand how vector similarity prevents duplicate actions across scans.
- **derived_fields expressions:** Functions like `fallback_chain()`, `inherit()`, `date_add()` are resolved at write time — find their implementations to understand the mini-expression language.
- **A different pathway type:** Trace `root.emails` (external source via Gmail API) or `root.family_events` (has `categories` and more complex extraction) for contrast.
