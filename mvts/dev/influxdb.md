---
description: >
  MVTS InfluxDB Reference — schema, queries, and operator timeline gap analysis across key MVTS tables.
  TRIGGER when: querying InfluxDB for MVTS data, diagnosing operator idle time, PPS starvation,
  predicted vs real time gaps, navigation leg analysis, bot assignment history.
  SKIP: live pod/log debugging — use /mvts/index, /mvts/ops/relay, or /mvts/ops/vtm-stuck instead.
---

# MVTS InfluxDB Reference

> **Engineer-only tool**: This is a technical reference for MVTS engineers.
> If you are an operations user, use `/mvts/index` instead and describe the problem in plain English.

All tables are in the `GreyOrange` database. Use `mcp__gor-global-mcp__query_metrics` with the environment alias.

Find the `InstallationId` for an environment:
```bash
kubectl exec mvts-0 -n <namespace> -- bash -c 'grep -i "installation\|influx" /app/data/config/local.application.properties'
```

---

## Tables

### `mvts_rtp_problem_assignment_details` — HTM assignments (written by MVTS)

One row per HTM bot assignment per solver cycle.

| Field | Description |
|-------|-------------|
| `assigned_bot_id` | Tag — HTM bot ID |
| `assigned_pps_id` | Tag — PPS the tote is going to |
| `msu_id` | Tote ID (`RELAYY_XXXX`) |
| `task_id` / `counter_id` | Task UUID and order reference |
| `assignment_type` | `new_task`, `cached`, or `current_schedule` |
| `bot_start_time` | When MVTS wants the bot to start moving (ms epoch) |
| `pps_queue_reach_time` | When MVTS predicts the bot reaches the PPS queue entrance (ms epoch) |
| `start_time` | When MVTS predicts the operator will start (ms epoch) |
| `end_time` | Predicted operator finish time (ms epoch) |
| `predicted_transit_time` | `pps_queue_reach_time - bot_start_time` in ms |
| `available_start_time` | When this bot is expected to be free for the next task |
| `problem_statement_time` / `request_id` | Identifies which solver cycle produced this row |
| `bins` / `entity_pick_sequence` | Bins to be picked and their sequence |

**Time relationships:**
```
bot_start_time ──transit──► pps_queue_reach_time ──queue wait──► start_time ──op time──► end_time
                 (predicted_transit_time)          (start - reach)             (end - start)
```

**Assignment types:**
| Type | Meaning |
|------|---------|
| `new_task` | Freshly assigned this cycle — GMC hasn't dispatched it yet |
| `cached` | Sent to GMC in a prior cycle, not yet acknowledged |
| `current_schedule` | GMC has dispatched it — bot is actively executing |

```influxql
SELECT * FROM mvts_rtp_problem_assignment_details
WHERE time > now() - 1h AND "InstallationId" = '<id>'
AND "assigned_bot_id" = '<bot_id>'
ORDER BY time DESC
```

**⚠️ `available_start_time` +24h bug (PPS starvation):**
If a `current_schedule` task has `bins = []` and `entity_pick_sequence = 0`, MVTS sets `available_start_time = problem_statement_time + 86400000ms`. Multiple bots showing +24h for the same PPS → PPS goes idle.

```influxql
SELECT assigned_bot_id, assignment_type, available_start_time, bins, entity_pick_sequence
FROM mvts_rtp_problem_assignment_details
WHERE time > now() - 1h AND "InstallationId" = '<id>'
AND "assigned_pps_id" = '<pps_id>'
AND assignment_type = 'current_schedule'
ORDER BY time DESC
```

---

### `mvts_pred_vs_real_times` — Predicted vs actual task timings (written by GMC)

One row per completed task. Tracks 4 legs end-to-end: **bot start → bot arrival at PPS → operator start → operator end**, with MVTS predicted and real timestamps for each.

**Tags** (use for filtering):
| Tag | Description |
|-----|-------------|
| `butler_id` | Bot ID |
| `pps_id` | Pick/Put Station |
| `fulfilment_area` | Warehouse zone |
| `installation_id` | Warehouse installation ID |
| `host` | MVTS pod hostname |

**Fields** — identifiers:
| Field | Description |
|-------|-------------|
| `butler_type` | Bot type (e.g. `htm`) |
| `butler_version` | Hardware version (e.g. `HTM_HAI_K50_L_E1`) |
| `transport_entity_id` | Tote/rack being transported |
| `task_key` | MVTS task UUID (join key with `mvts_rtp_problem_assignment_details.task_id`) |
| `reached_to_rack` | Epoch ms when bot physically reached the rack |
| `lifted_rack` | Epoch ms when bot lifted the rack |

**Fields** — the 4 predicted vs real leg pairs (all epoch ms):

| Milestone | Predicted | Real |
|-----------|-----------|------|
| Bot starts moving toward PPS | `mvts_predicted_ranger_start_time` | `real_ranger_start_time` |
| Bot arrives at PPS dock | `mvts_predicted_bot_arrival_time` | `real_bot_arrival_time` |
| Operator starts picking | `mvts_predicted_operator_start_time` | `real_operator_start_time` |
| Operator finishes | `mvts_predicted_operator_end_time` | `real_operator_end_time` |
| Bot free for next task | `gmc_predicted_ranger_available_time` | `real_ranger_available_time` |

**Timeline:**
```
bot_start ──transit──► bot_arrival ──dock_wait──► op_start ──op_duration──► op_end ──► bot_free
```

**Derived metrics** (compute in query or post-process):
| Metric | Formula | What it means |
|--------|---------|---------------|
| Transit time diff | `(real_arrival - real_start) - (pred_arrival - pred_start)` | Positive = bot slower than predicted |
| Bot start delay | `real_ranger_start_time - pred_ranger_start_time` | Positive = bot dispatched late |
| PPS dock wait | `real_operator_start_time - real_bot_arrival_time` | How long bot waited at dock before operator started |
| Operator duration diff | `(real_op_end - real_op_start) - (pred_op_end - pred_op_start)` | Positive = operator took longer than predicted |

**⚠️ Known behavior in stg001:** `real_bot_arrival_time` is null for all rows — the bot-arrival event at PPS is not being recorded. Operator start/end are written correctly. Confirmed working in stpbulk. Check MVTS version or whether tasks complete delivery in that env.

---

#### Query: Full per-task deviation across all legs

```influxql
SELECT butler_id, pps_id, transport_entity_id, task_key,
  (real_ranger_start_time - mvts_predicted_ranger_start_time) / 1000 AS bot_start_delay_s,
  ((real_bot_arrival_time - real_ranger_start_time) - (mvts_predicted_bot_arrival_time - mvts_predicted_ranger_start_time)) / 1000 AS transit_diff_s,
  (real_operator_start_time - real_bot_arrival_time) / 1000 AS dock_wait_s,
  ((real_operator_end_time - real_operator_start_time) - (mvts_predicted_operator_end_time - mvts_predicted_operator_start_time)) / 1000 AS op_duration_diff_s
FROM mvts_pred_vs_real_times
WHERE time > now() - 1h AND "installation_id" = '<id>'
ORDER BY time DESC
```

#### Query: Find tasks where bot arrived much later than predicted (transit overrun > 60s)

```influxql
SELECT butler_id, pps_id, transport_entity_id,
  ((real_bot_arrival_time - real_ranger_start_time) - (mvts_predicted_bot_arrival_time - mvts_predicted_ranger_start_time)) / 1000 AS transit_diff_s
FROM mvts_pred_vs_real_times
WHERE time > now() - 1h AND "installation_id" = '<id>'
  AND (real_bot_arrival_time - real_ranger_start_time) - (mvts_predicted_bot_arrival_time - mvts_predicted_ranger_start_time) > 60000
ORDER BY time DESC
```

#### Query: Find tasks where operator took much longer than predicted (operator overrun > 30s)

```influxql
SELECT butler_id, pps_id, transport_entity_id,
  (real_operator_end_time - real_operator_start_time) / 1000 AS actual_op_duration_s,
  (mvts_predicted_operator_end_time - mvts_predicted_operator_start_time) / 1000 AS predicted_op_duration_s,
  ((real_operator_end_time - real_operator_start_time) - (mvts_predicted_operator_end_time - mvts_predicted_operator_start_time)) / 1000 AS op_overrun_s
FROM mvts_pred_vs_real_times
WHERE time > now() - 1h AND "installation_id" = '<id>'
  AND (real_operator_end_time - real_operator_start_time) - (mvts_predicted_operator_end_time - mvts_predicted_operator_start_time) > 30000
ORDER BY time DESC
```

#### Query: Per-PPS avg transit deviation (find which stations are worst)

```influxql
SELECT mean(((real_bot_arrival_time - real_ranger_start_time) - (mvts_predicted_bot_arrival_time - mvts_predicted_ranger_start_time)) / 1000) AS avg_transit_diff_s
FROM mvts_pred_vs_real_times
WHERE time > now() - 6h AND "installation_id" = '<id>'
GROUP BY "pps_id"
```

---

**Diagnosing by deviation pattern:**

| Pattern | Root cause |
|---------|-----------|
| `bot_start_delay_s` large positive | Bot was dispatched late — check `available_start_time` in prior `mvts_rtp_problem_assignment_details` cycle |
| `transit_diff_s` large positive | Bot took longer to travel — drill into `task_cycle_times` nav legs for deadlocks or floor congestion |
| `transit_diff_s` large negative | Bot assigned from much closer than predicted — MVTS over-estimated travel distance |
| `dock_wait_s` large | PPS dock was occupied when bot arrived — another bot was still at the station |
| `op_duration_diff_s` large positive | Operator slower than predicted — check if bins/items count was higher than expected |
| Operator start systematically early (avg -40s) | MVTS over-predicts transit time; bots arrive earlier than expected (observed in stpbulk) |
| `real_bot_arrival_time` null | Bot-arrival event not firing — task may be cancelled before PPS delivery, or MVTS version missing the write |

---

### `task_cycle_times` — Actual bot movement times (written by Navigation)

One row per physical bot movement (goto completion).

| Field | Description |
|-------|-------------|
| `butler_id` | Bot ID (field, not tag) |
| `idc_time_ms` | IDC predicted travel time |
| `path_travel_time_ms` | Actual travel time |
| `deadlock_counts` | Navigation-level path conflicts |
| `deadlock_resolution_time_ms` | Time lost to navigation deadlocks |
| `start` / `goal` | Source and destination coordinates |
| `path_type` | Movement type (see leg sequence below) |
| `original_idc_time_ms` | IDC time before multiplier adjustment |

Full journey leg sequence:
```
relay_storable_to_relay_io_point    — pick up tote from rack
relay_io_point_to_relay_io_point    — cross-aisle transit
relay_io_point_to_pps_entry         — approach PPS
pps_entry_to_pps                    — enter PPS queue / dock
pps_to_pps_exit                     — exit after operator picks
pps_exit_to_relay_io_point          — return to relay area
```

```influxql
SELECT path_type, idc_time_ms, path_travel_time_ms, deadlock_counts, start, goal
FROM task_cycle_times
WHERE time > now() - 1h AND "installation_id" = '<id>'
AND butler_id = <bot_id>
ORDER BY time ASC
```

---

### `mvts_relay_vtm_bot_assignment_details` — VTM assignments

| Field | Description |
|-------|-------------|
| `bot_id` | VTM bot ID (field, integer) |
| `current_aisle` | Aisle the bot is currently in |
| `task_chain` | Full pick/drop sequence |
| `request_id` | Solver cycle ID |
| `InstallationId` | Tag |

---

### `mvts_relay_vtm_bot_aisle_switches` — VTM aisle changes

| Field | Description |
|-------|-------------|
| `bot_id` | VTM bot ID (field, integer) |
| `bot_id_tag` | Tag for filtering |
| `from_aisle` | Previous aisle (negative = dummy/no aisle) |
| `to_aisle` | New assigned aisle |
| `transitive_aisles` | Intermediate aisles in path |
| `request_id` | Solver cycle that triggered the switch |

---

## Cross-Table: Operator Timeline Gap Analysis

**Join key**: `mvts_rtp_problem_assignment_details.task_id` == `mvts_pred_vs_real_times.task_key` (GMC UUIDs).
`task_cycle_times` uses a Navigation UUID — correlate by `butler_id + time window` instead.

### Step 1 — Find tasks with large gaps

| Gap | Formula | Meaning |
|-----|---------|---------|
| Bot start delay | `real_ranger_start_time - mvts_predicted_ranger_start_time` | MVTS underestimated bot busy time |
| Transit time diff | `(real_arrival - real_start) - (pred_arrival - pred_start)` | Net travel time deviation (removes start delay from transit) |
| Transit overrun (absolute) | `real_bot_arrival_time - mvts_predicted_bot_arrival_time` | Raw arrival lateness — combines start delay + travel time |
| PPS dock wait | `real_operator_start_time - real_bot_arrival_time` | Bot arrived but dock wasn't free (another bot at station) |
| Operator duration overrun | `(real_op_end - real_op_start) - (pred_op_end - pred_op_start)` | Operator took longer than planned |
| Operator start drift | `real_operator_start_time - mvts_predicted_operator_start_time` | Negative = operator started early (bot arrived early) |

```influxql
SELECT task_key, butler_id, pps_id,
  (real_bot_arrival_time - mvts_predicted_bot_arrival_time) AS transit_overrun_ms,
  (real_operator_start_time - real_bot_arrival_time) AS pps_queue_wait_ms,
  (real_operator_end_time - real_operator_start_time) AS actual_operator_time_ms,
  (mvts_predicted_operator_end_time - mvts_predicted_operator_start_time) AS predicted_operator_time_ms
FROM mvts_pred_vs_real_times
WHERE time > now() - 1h AND "installation_id" = '<id>'
ORDER BY time DESC
```

**Diagnosing by assignment_type gap:**
- Gap between two `current_schedule`/`cached` rows → navigation or execution problem
- Gap between two `new_task` rows → MVTS planning problem

### Step 2 — Drill into navigation legs

```influxql
SELECT path_type, start, goal, idc_time_ms, path_travel_time_ms,
       deadlock_counts, deadlock_resolution_time_ms
FROM task_cycle_times
WHERE time > '<bot_start_time RFC3339>' AND time < '<start_time + 10min RFC3339>'
AND "installation_id" = '<id>'
AND butler_id = <bot_id>
ORDER BY time ASC
```

### Step 3 — Interpret slow legs

| Pattern | Root cause |
|---------|-----------|
| `relay_io_point_to_relay_io_point` slow | Floor congestion on cross-aisle transit |
| `pps_entry_to_pps` actual >> idc, `idc ≈ 0` | PPS queue wait — another bot at dock |
| `deadlock_counts > 0`, high `deadlock_resolution_time_ms` | Navigation-level deadlock |
| All legs normal but bot arrived late | MVTS dispatched too late — check `available_start_time` from prior cycle |
| `pps_exit_to_relay_io_point` slow | Return journey congested |
