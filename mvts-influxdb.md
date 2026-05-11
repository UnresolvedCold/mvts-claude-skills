---
description: >
  MVTS InfluxDB Reference — schema, queries, and operator timeline gap analysis across key MVTS tables.
  TRIGGER when: querying InfluxDB for MVTS data, diagnosing operator idle time, PPS starvation,
  predicted vs real time gaps, navigation leg analysis, bot assignment history.
  SKIP: live pod/log debugging — use mvts-debug, mvts-relay-debug, or mvts-vtm-deadlock instead.
---

# MVTS InfluxDB Reference

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

One row per completed task.

| Field | Description |
|-------|-------------|
| `mvts_predicted_ranger_start_time` | When MVTS predicted the bot would start |
| `mvts_predicted_bot_arrival_time` | When MVTS predicted bot arrival at rack |
| `mvts_predicted_operator_start/end_time` | MVTS predicted operator window |
| `real_ranger_start_time` | When bot actually started |
| `real_bot_arrival_time` | When bot actually arrived at rack |
| `real_operator_start/end_time` | Actual operator pick window |
| `butler_id` | Tag — bot ID |
| `pps_id` | Tag — PPS ID |

```influxql
SELECT (real_ranger_start_time - mvts_predicted_ranger_start_time) AS start_delay_ms,
       (real_bot_arrival_time - mvts_predicted_bot_arrival_time) AS transit_overrun_ms,
       *
FROM mvts_pred_vs_real_times
WHERE time > now() - 1h AND "installation_id" = '<id>'
ORDER BY time DESC
```

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
| Transit overrun | `real_bot_arrival_time - mvts_predicted_bot_arrival_time` | Bot took longer to travel |
| PPS queue wait | `real_operator_start_time - real_bot_arrival_time` | Bot arrived but dock wasn't free |
| Operator overrun | `real_operator_end_time - mvts_predicted_operator_end_time` | Operator took longer than planned |

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
