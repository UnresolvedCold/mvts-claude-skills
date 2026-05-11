# MVTS Operator Time Audit

Audits how `serviced_orders` and `operator_time` values inflate or deflate for a specific task in a bot's `ranger_schedule` across problem statements.

**Use when**: operator_time values in `serviced_orders` are unexpectedly high/low, fluctuating, dropping to 0, or you want to trace when orders were batched in/removed for a task.

---

## Step 1 — Resolve environment and namespace

```bash
# List environments
mcp__gor-global-mcp__list_environments

# If on-prem, find the namespace
ssh JumpServer "kubectl get namespaces | grep -i '<env_name>'"
```

Namespace pattern: `stpbulk-cluster-<env>-greymatter`

---

## Step 2 — Find which log files contain the task

The task UUID appears in `Message:` lines (problem statements). Search archived logs:

```bash
# Search recent days first (adjust date range as needed)
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c '
zgrep -l \"<task_uuid>\" /app/data/logs/scheduler.YYYY-MM-DD.*.log.gz 2>/dev/null | sort
'"
```

If not in the live log (`scheduler.log`), work backward through dates. The UUID will appear once per PS cycle (~every 30s) while the task is alive.

---

## Step 3 — Extract serviced_orders across all matching PSes

Write and run this Python script inside the pod:

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c 'cat > /tmp/audit_op_time.py << PYEOF
import gzip, json, re

task_id = \"<task_uuid>\"
bot_id = <bot_id>
# List all matching log files from Step 2
files = [\"/app/data/logs/scheduler.<DATE>.<N>.log.gz\"]

for f in files:
    with gzip.open(f, \"rt\", errors=\"replace\") as fh:
        for line in fh:
            if \"Message:\" not in line or task_id not in line:
                continue
            ts_match = re.match(r\"(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d+Z)\", line)
            timestamp = ts_match.group(1) if ts_match else \"?\"
            idx = line.index(\"Message:\") + len(\"Message:\")
            json_str = line[idx:].strip()
            try:
                ps = json.loads(json_str)
            except:
                continue
            request_id = ps.get(\"request_id\", \"?\")
            for ranger in ps.get(\"ranger_list\", []):
                if ranger.get(\"id\") == bot_id:
                    for task in ranger.get(\"ranger_schedule\", []):
                        if task.get(\"task_key\") == task_id:
                            so = task.get(\"serviced_orders\", [])
                            print(json.dumps({\"ts\": timestamp, \"request_id\": request_id, \"serviced_orders\": so}))
                    break
PYEOF
echo wrote'"

ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- python3 /tmp/audit_op_time.py"
```

---

## Step 4 — Interpret the output

Each JSON line is one PS cycle. Parse and look for:

| Pattern | Meaning |
|---------|---------|
| `serviced_orders` count increases | New orders batched into this task (PPS batching) |
| `operator_time` drops to 0 for some orders | Those orders were already picked / being processed by operator |
| All `operator_time` = 0 | Task is at the PPS, operator is finishing |
| `serviced_orders` becomes `[]` | Task completed — GMC dispatched and cleared it |
| `operator_time` jumps up | New or re-estimated orders added mid-task |

**Sum of operator_time** is what MVTS uses to estimate how long the operator will spend on this bot's tote. If this inflates unexpectedly, MVTS will overestimate `end_time` and delay the next bot assignment for that PPS.

**Key fields per order:**
- `order_id` — GMC order identifier
- `operator_time` — ms MVTS estimates for this order's pick
- `order_priority` — MEDIUM/HIGH/CRITICAL
- `is_order_critical` — boolean
- `order_pbt` — epoch ms of promised-by time
- `order_allocation_time` — when the order was allocated to this task

---

## Common root causes for inflation/deflation

| Symptom | Likely cause |
|---------|-------------|
| operator_time inflates PS-over-PS | Orders being added to the batch; check if batching threshold is too large |
| operator_time suddenly → 0 for most orders | Operator already scanning/picking them; normal mid-task behavior |
| operator_time stays at 0 for many PS cycles | MVTS has a stale `current_schedule` task with no real bin data — check `bins=[]` / `entity_pick_sequence=0` which causes `available_start_time = now + 24h` bug |
| Count drops (orders removed) | Order cancelled or re-allocated to another task |
| `serviced_orders` cleared abruptly | Task dispatched to GMC and completed |

**The +24h bug**: If `bins=[]` and `entity_pick_sequence=0` on a `current_schedule` task, MVTS sets `available_start_time = problem_statement_time + 86400000ms`. This starves the PPS of new assignments. Fix: investigate why GMC is not reporting real bin data for this task.

---

## Notes

- Task UUID (`task_key`) is stable across PS cycles for the lifetime of the task.
- `ranger_schedule` entries for a bot reflect both pending and in-progress assignments.
- The task status field in ranger_schedule may be `null` — use `assignment_type` from `mvts_rtp_problem_assignment_details` InfluxDB table to distinguish `new_task` / `cached` / `current_schedule`.
- Cross-reference with `mvts_rtp_problem_assignment_details` using `task_id` to get predicted/actual times.
