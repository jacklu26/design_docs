# Node Budget Controller - Design Doc

## Problem Statement

Nodes in the famgraph process data through policies (LLM-powered ingestion/extraction pipelines). Each policy invocation incurs LLM costs tracked per `node_path` in ClickHouse. Today there is no mechanism to cap spending on a per-node basis -- a misconfigured or unexpectedly expensive node can run unchecked. We need a budget controller that automatically deactivates nodes when their cost exceeds a configured threshold for a given period, and reactivates them when the next period begins.

## Current Architecture

### Graph Loading and Processing

```
famgraph.json / DB  -->  GraphEngine  -->  Node tree with policies
                                                |
                                          get_pathways()
                                                |
                                          policy.active?
                                           /        \
                                         Yes         No
                                          |           |
                                       Pathway     Skipped
                                          |
                                    PagingWorkflow (LLM calls)
                                          |
                                    LLM call with node_path
                                          |
                                    ClickHouse llm_calls table
```

### Key facts

- `Policy` has `active: bool = True` (fambot/src/memory/policy.py, line 430). Checked in 6 places across injected_models.py and fam_service.py.
- `Node` does NOT have an `active` field (fambot/src/memory/models.py, lines 176-206).
- Cost is tracked per `node_path` in ClickHouse `llm_calls` table with `cost_in_dollars`.
- Periodic workflows use Temporal Schedules (see Chronos in src/workflows/chronos.py and src/run_worker.py).
- Famgraph specs can be stored per-user in DB (`famgraph` + `famgraph_uid` tables) or loaded from the default JSON file.
- There is no runtime mutation of famgraph today. Writes only happen via `sync_famgraphs_to_db()`.

---

## Design

### Overview

```
Budget Controller Workflow (runs every N minutes)
    |
    |-- Query ClickHouse: cost per node_path in current period
    |-- Query Postgres: node_budget configs
    |-- Compare spend vs budget
    |       |
    |       |-- Over budget --> Write node_budget.is_exceeded = True
    |       |-- New period  --> Reset is_exceeded, period_spend
    |
    v
GraphEngine Init (on next request)
    |
    |-- Load famgraph spec
    |-- Query node_budget for exceeded entries
    |-- Set node.active = False on exceeded nodes
```

---

### 1. Add `active` field to `Node`

**File:** fambot/src/memory/models.py

Add `active: bool = True` to the `Node` class. This is a general-purpose feature -- useful for manually disabling nodes in famgraph.json too, independent of budgets.

**Effect:** When `node.active = False`, ALL policies on that node are skipped regardless of their individual `active` flags.

**Enforcement points** (all in fambot/src/memory/injected_models.py):

| Method                     | Line  | Change                                                    |
|----------------------------|-------|-----------------------------------------------------------|
| `get_pathways()`           | ~2193 | Add `if not node.active: continue` before policy loop     |
| `get_nodes_with_category()`| ~1002 | Add `if not node.active: continue` before policy loop     |
| `get_activities()`         | ~2518 | Add `if not node.active: return []` after node lookup     |
| `get_activity()`           | ~2581 | Add `if not node.active: return None` after node lookup   |
| `get_flow()`               | ~2662 | Add `if not node.active: return None` after node lookup   |

And in fambot/src/fam/fam_service.py:

| Method         | Line  | Change                                           |
|----------------|-------|--------------------------------------------------|
| `ops_health()` | ~1913 | Check `node.active` before checking policies     |

---

### 2. Create `node_budget` entity

**File:** fambot/src/memory/famgraph.json (schema section)

New entity in the `schema` block:

```json
"node_budget": {
    "_extends_from": {
        "bases": ["system_base"]
    },
    "_uniq_constraints": {
        "_uniq_node_path": ["node_path"]
    },
    "id": {
        "type": "integer",
        "primary_key": true,
        "autoincrement": true
    },
    "node_path": {
        "type": "string",
        "index": true,
        "description": "Graph node path (e.g. root.family_activities)"
    },
    "budget_usd": {
        "type": "float",
        "description": "Maximum spend allowed per period in USD"
    },
    "period": {
        "type": "string",
        "default": "'daily'",
        "description": "Budget period: daily, weekly, or monthly"
    },
    "period_start": {
        "type": "timestamp",
        "nullable": true,
        "description": "Start of the current budget period"
    },
    "period_spend_usd": {
        "type": "float",
        "default": "0.0",
        "description": "Accumulated spend in current period (from ClickHouse)"
    },
    "is_exceeded": {
        "type": "boolean",
        "default": "false",
        "description": "True when spend has exceeded budget this period"
    },
    "active": {
        "type": "boolean",
        "default": "true",
        "description": "Whether this budget rule is active"
    }
}
```

After adding to the schema, run:

- `sync_spec_to_db.py` to generate the SQLAlchemy model
- Alembic autogenerate to create the migration

---

### 3. Add `node_budget` to the graph

**File:** fambot/src/memory/famgraph.json (graph section)

Add a node under the graph for admin access to budget configs:

```json
"node_budgets": {
    "entity": "node_budget",
    "description": "Budget limits per graph node for cost control"
}
```

---

### 4. GraphEngine: apply budget overrides at init

**File:** fambot/src/memory/injected_models.py

After `__inject__()` builds the graph, query `node_budget` for any rows where `is_exceeded = True` and `active = True`, then set `node.active = False` on matching nodes.

```python
def _apply_budget_overrides(self):
    """Deactivate nodes that have exceeded their budget."""
    try:
        exceeded = self.engine.get_exceeded_node_budgets()
        for row in exceeded:
            node = self.graph.all_nodes.get(row["node_path"])
            if node is not None:
                node.active = False
                logger.warning(
                    f"Node {row['node_path']} deactivated: "
                    f"${row['period_spend_usd']:.2f} / ${row['budget_usd']:.2f} budget"
                )
    except Exception as e:
        logger.error(f"Failed to apply budget overrides: {e}")
```

Call this at the end of `__init__`, after `__inject__` completes.

**File:** fambot/src/db/sql/engine.py

Add a new method:

```python
def get_exceeded_node_budgets(self) -> list[dict]:
    """Return node_budget rows where is_exceeded=True and active=True."""
    table = db_metadata.tables["node_budget"]
    stmt = select(table).where(
        and_(table.c.is_exceeded == True, table.c.active == True)
    )
    return self.execute_select(stmt)
```

---

### 5. BudgetControllerWorkflow

**New files:**

- `src/workflows/budget_controller.py` -- workflow definition
- `src/models/budget_controller.py` -- request dataclass
- `src/activities/budget_controller.py` -- activities

#### Request model

```python
@dataclass
class BudgetControllerRequest:
    version: str = ""
```

#### Activities

Two activities:

**`check_node_budgets_activity`** -- the core logic:

1. Query Postgres for all active `node_budget` rows.
2. For each, determine the time window based on `period` and `period_start`.
3. Query ClickHouse for cost in that window:

```sql
SELECT
    node_path,
    sum(toFloat64(cost_in_dollars)) AS total_cost
FROM llm_calls
WHERE timestamp >= {period_start}
  AND node_path = {node_path}
GROUP BY node_path
```

4. Compare `total_cost` against `budget_usd`.
5. If exceeded: UPDATE `node_budget` SET `is_exceeded = True`, `period_spend_usd = total_cost`.
6. Clear GraphEngine cache for affected users so next load picks up the override.

**`rollover_budgets_activity`** -- period management:

1. Query all `node_budget` rows where `period_start + period_duration < now()`.
2. For those: UPDATE SET `is_exceeded = False`, `period_spend_usd = 0`, `period_start = now()`.
3. Clear GraphEngine cache.

#### Workflow

```python
@workflow.defn
class BudgetControllerWorkflow:
    @workflow.run
    async def run(self, request: BudgetControllerRequest) -> None:
        # First: roll over any expired periods
        await workflow.execute_activity(
            rollover_budgets_activity,
            start_to_close_timeout=timedelta(seconds=60),
            retry_policy=RetryPolicy(maximum_attempts=3),
        )
        # Then: check current period budgets
        await workflow.execute_activity(
            check_node_budgets_activity,
            start_to_close_timeout=timedelta(seconds=60),
            retry_policy=RetryPolicy(maximum_attempts=3),
        )
```

---

### 6. Schedule the workflow

**File:** src/run_worker.py

Follow the Chronos pattern:

```python
BUDGET_CONTROLLER_INTERVAL_SECONDS = int(
    os.environ.get("BUDGET_CONTROLLER_INTERVAL_SECONDS", "600")  # 10 minutes
)

async def start_budget_controller(client):
    schedule_id = "budget-controller"
    try:
        handle = client.get_schedule_handle(schedule_id)
        await handle.delete()
    except Exception:
        pass
    await client.create_schedule(
        schedule_id,
        schedule=Schedule(
            action=ScheduleActionStartWorkflow(
                BudgetControllerWorkflow.run,
                id="budget_controller_workflow",
                task_queue=GENESIS_TASK_QUEUE,
                arg=BudgetControllerRequest(),
                execution_timeout=timedelta(minutes=5),
            ),
            spec=ScheduleSpec(
                intervals=[
                    ScheduleIntervalSpec(
                        every=timedelta(seconds=BUDGET_CONTROLLER_INTERVAL_SECONDS)
                    )
                ],
            ),
            policy=SchedulePolicy(overlap=ScheduleOverlapPolicy.SKIP),
        ),
    )
```

Register in the Worker:

- Add `BudgetControllerWorkflow` to `workflows=[...]`
- Add `check_node_budgets_activity` and `rollover_budgets_activity` to `activities=[...]`
- Call `await start_budget_controller(client)` during startup

---

## Data Flow Summary

```
BudgetController (every 10m)
    |
    |----> Postgres: Read active node_budget rows
    |----> ClickHouse: Query cost per node_path in current period
    |----> Compare spend vs budget
    |
    |  [Over budget]
    |----> Postgres: SET is_exceeded = True
    |----> GraphEngine: Invalidate cache
    |
    |  [Period expired]
    |----> Postgres: Reset is_exceeded, period_spend, period_start
    |----> GraphEngine: Invalidate cache
    |
    v
GraphEngine (next request for any user)
    |
    |----> Postgres: Load famgraph + query node_budget
    |----> Apply overrides: node.active = False for exceeded
    |----> get_pathways() skips inactive nodes
```

---

## Files Changed (Summary)

| File                                      | Change                                                     |
|-------------------------------------------|------------------------------------------------------------|
| fambot/src/memory/models.py               | Add `active: bool = True` to `Node`                       |
| fambot/src/memory/famgraph.json           | Add `node_budget` entity to schema; add `node_budgets` node to graph |
| fambot/src/memory/injected_models.py      | Add `_apply_budget_overrides()` to GraphEngine; add `node.active` checks in 5 methods |
| fambot/src/db/sql/engine.py               | Add `get_exceeded_node_budgets()` method                   |
| fambot/src/fam/fam_service.py             | Update `ops_health()` to check `node.active`               |
| src/workflows/budget_controller.py        | NEW -- BudgetControllerWorkflow                            |
| src/models/budget_controller.py           | NEW -- BudgetControllerRequest                             |
| src/activities/budget_controller.py       | NEW -- check + rollover activities                         |
| src/run_worker.py                         | Register workflow, activities, and schedule                 |
| Alembic migration                         | NEW -- auto-generated for `node_budget` table              |

---

## Implementation Order

1. **Node.active field + enforcement** -- smallest change, immediately useful for manual disabling
2. **node_budget entity** -- schema + migration
3. **GraphEngine budget override** -- wires node_budget to node.active
4. **BudgetControllerWorkflow + activities** -- the periodic checker
5. **Schedule registration** -- wire it into run_worker.py
6. **Tests** -- graph topology tests for node.active, unit tests for budget check logic

---

## Open Questions

1. **Budget granularity:** This design uses global per-node budgets (one budget per `node_path` across all users). Should we support per-user overrides in the future?
2. **Alerting:** Should the budget controller send notifications (email, Slack) when a node is deactivated, or is logging sufficient?
3. **Soft limits:** Should there be a warning threshold (e.g. 80% of budget) that logs/alerts without deactivating?
4. **Manual override:** Should there be an admin API to force-reactivate a node mid-period despite budget exceedance?
5. **ClickHouse query cost:** The budget controller queries ClickHouse every 10 minutes for each budgeted node. If we have many budgeted nodes, consider batching into a single query with GROUP BY.
