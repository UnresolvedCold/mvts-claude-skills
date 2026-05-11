---
description: >
  MVTS IDC Multiplier Validation — validates whether MVTS computed the correct dynamic IDC multiplier (k)
  for a given leg type and time window by recomputing grid-optimal k from raw task_cycle_times data
  and comparing against what MVTS stored in mvts_multiplier_msu_to_pps (or equivalent table).
  TRIGGER when: user asks to validate IDC multiplier, check if k value is correct, investigate why
  MVTS is over/under-estimating transit times, debug IDC congestion factor, or audit
  ENABLE_PERIODIC_IDC_MULTIPLIER_CALCULATION behavior.
  SKIP: general IDC path calculation questions not related to the multiplier; questions about infinite IDC.
argument-hint: Optional environment name and time window (e.g. "aphrelaybulk last 15 min")
allowed-tools: ["Bash", "mcp__gor-global-mcp__list_environments", "mcp__gor-global-mcp__kube_connect",
  "mcp__gor-global-mcp__kube_shell", "mcp__gor-global-mcp__query_metrics",
  "mcp__gor-global-mcp__search_influxql_catalog", "mcp__gor-global-mcp__describe_measurement"]
---

# MVTS IDC Multiplier Validation

Validates whether MVTS computed the correct dynamic IDC congestion multiplier (k) by recomputing
the grid-optimal k from raw `task_cycle_times` data and comparing with what MVTS actually stored.

---

## Background

When `ENABLE_PERIODIC_IDC_MULTIPLIER_CALCULATION=true`, MVTS runs a grid search every solver cycle
to find the best k (congestion multiplier) that minimizes MAE of:

```
k × original_idc_time ≈ total_path_travel_time
```

Grid candidates are controlled by:
- `IDC_MULTIPLIER_START` (default: 1.0)
- `IDC_MULTIPLIER_END` (default: 5.0)
- `IDC_MULTIPLIER_STEP` (default: 0.25)
→ 17 candidates: {1.0, 1.25, 1.5, ..., 5.0}

**Known bug**: MVTS's query filters on `event_name='goto_completed'` as a **field** (not a tag),
which drastically reduces the sample count — sometimes from hundreds of rows to just 1 row in a
15-minute window. This makes k unreliable and causes MVTS to mis-estimate transit times → high TPQ.

---

## Step 1 — Resolve environment and verify feature is enabled

```bash
# Resolve namespace from alias
mcp__gor-global-mcp__list_environments(query="<env_alias>")

# For stpbulk environments, check config via SSH:
ssh JumpServer "kubectl exec mvts-0 -n stpbulk-cluster-<env>-greymatter -- bash -c \
  'grep -E \"IDC_MULTIPLIER|ENABLE_PERIODIC\" /app/data/config/local.application.properties'"
```

Expected config:
```
ENABLE_PERIODIC_IDC_MULTIPLIER_CALCULATION=true
IDC_MULTIPLIER_START=1.0
IDC_MULTIPLIER_END=5.0
IDC_MULTIPLIER_STEP=0.25
```

If `ENABLE_PERIODIC_IDC_MULTIPLIER_CALCULATION=false`, the multiplier is static — skip to Step 5
to check what static value is in use.

---

## Step 2 — Get the MVTS query time window

MVTS uses a 15-minute sliding window when computing k. To replicate it, use the same window.
Get the current epoch time in nanoseconds (InfluxDB format):

```bash
python3 -c "import time; t=int(time.time()*1e9); print(f'now={t}'); print(f'15min ago={t - 15*60*int(1e9)}')"
```

Or use a specific historical window if investigating a past incident:
- Convert ISO timestamp to epoch ns: `python3 -c "from datetime import datetime; print(int(datetime(2026,5,10,12,0).timestamp()*1e9))"`

---

## Step 3 — Query raw task_cycle_times (MSU_TO_PPS leg types)

The MSU_TO_PPS leg covers all path types where a bot carries a tote from storage/relay to a PPS.

**For stpbulk environments** (InfluxDB at `172.29.28.82:8086`, not accessible via gor-global-mcp):

Write the query to a script and copy it in:
```bash
cat > /tmp/k_query.sh << 'SCRIPT'
#!/bin/bash
NS="stpbulk-cluster-<env>-greymatter"
INFLUX="http://172.29.28.82:8086"
DB="GreyOrange"
# Time window in nanoseconds (epoch)
T_START=<start_ns>
T_END=<end_ns>

Q="SELECT original_idc_time, total_path_travel_time, pps_id
   FROM task_cycle_times
   WHERE (original_path_type = 'storable_to_pps_queue'
      OR original_path_type = 'relay_io_point_to_pps_entry'
      OR original_path_type = 'relay_io_point_to_pps'
      OR original_path_type = 'non_storable_to_relay_io_point'
      OR original_path_type = 'non_storable_to_pps'
      OR original_path_type = 'relay_io_point_to_conveyor_entry'
      OR original_path_type = 'relay_storable_to_conveyor_entry'
      OR original_path_type = 'storable_to_relay_io_point')
   AND time > ${T_START}
   AND time < ${T_END}
   AND installation_id = 'stpbulk-<env>'
   GROUP BY pps_id"

kubectl exec mvts-0 -n $NS -- bash -c \
  "curl -sG '${INFLUX}/query' \
    --data-urlencode 'db=${DB}' \
    --data-urlencode 'q=${Q}'"
SCRIPT
scp /tmp/k_query.sh JumpServer:/tmp/k_query.sh
ssh JumpServer "kubectl cp /tmp/k_query.sh stpbulk-cluster-<env>-greymatter/mvts-0:/tmp/k_query.sh && \
               kubectl exec mvts-0 -n stpbulk-cluster-<env>-greymatter -- bash /tmp/k_query.sh" \
  > /tmp/k_data.json
```

**⚠️ Do NOT add `AND event_name = 'goto_completed'` to this query.** That is the MVTS bug —
it filters out most rows because `event_name` is a field, not a tag. Our validation uses the full
sample set.

**For qa4 environments** (accessible via gor-global-mcp):
```
mcp__gor-global-mcp__query_metrics(
  environment="<env>",
  measurement="task_cycle_times",
  ...
)
```

---

## Step 4 — Compute grid-optimal k per PPS

Parse the InfluxDB JSON response and run MAE minimization:

```python
# Save the InfluxDB response to /tmp/k_data.json first, then run:
cat > /tmp/compute_k.py << 'EOF'
import json, sys
import numpy as np

with open('/tmp/k_data.json') as f:
    data = json.load(f)

# Grid from MVTS config
K_START = 1.0
K_END   = 5.0
K_STEP  = 0.25
grid = [round(K_START + i * K_STEP, 10) for i in range(int((K_END - K_START) / K_STEP) + 1)]

results = []
for series in data.get('results', [{}])[0].get('series', []):
    pps_id = series.get('tags', {}).get('pps_id', 'unknown')
    cols = series['columns']
    rows = series['values']
    idc_col = cols.index('original_idc_time')
    tpt_col = cols.index('total_path_travel_time')

    samples = [(r[idc_col], r[tpt_col]) for r in rows
               if r[idc_col] is not None and r[tpt_col] is not None and r[idc_col] > 0]

    if not samples:
        print(f"PPS {pps_id}: no valid samples")
        continue

    idcs = [s[0] for s in samples]
    tpts = [s[1] for s in samples]

    best_k, best_mae = None, float('inf')
    for k in grid:
        mae = sum(abs(k * idc - tpt) for idc, tpt in samples) / len(samples)
        if mae < best_mae:
            best_mae, best_k = mae, k

    results.append((pps_id, len(samples), best_k, best_mae))
    print(f"PPS {pps_id:>6}  n={len(samples):>3}  best_k={best_k:.2f}  MAE={best_mae:.3f}s")

print(f"\nSummary: {len(results)} PPSes analyzed")
EOF
python3 /tmp/compute_k.py
```

---

## Step 5 — Query what MVTS actually computed

**Table**: `mvts_multiplier_msu_to_pps` (written by MVTS after each grid search)

| Field/Tag | Description |
|-----------|-------------|
| `pps_id` | Tag — PPS ID |
| `congestion` | Field — the k value MVTS picked |
| `InstallationId` | Tag |

```influxql
SELECT * FROM mvts_multiplier_msu_to_pps
WHERE time > <T_START_ns> AND time < <T_END_ns>
AND "InstallationId" = '<installation_id>'
GROUP BY pps_id
ORDER BY time DESC
LIMIT 1
```

For stpbulk:
```bash
ssh JumpServer "kubectl exec mvts-0 -n stpbulk-cluster-<env>-greymatter -- bash -c \
  \"curl -sG http://172.29.28.82:8086/query \
    --data-urlencode 'db=GreyOrange' \
    --data-urlencode 'q=SELECT congestion FROM mvts_multiplier_msu_to_pps WHERE time > <T_START_ns> AND \\\"InstallationId\\\"=\\\"<id>\\\" GROUP BY pps_id ORDER BY time DESC LIMIT 1'\""
```

---

## Step 6 — Compare and interpret

Build a side-by-side table:

| PPS | n (samples) | Best k (grid) | MVTS k | Match? | MAE@best | MAE@MVTS |
|-----|-------------|---------------|--------|--------|----------|----------|
| ... | ...         | ...           | ...    | ✅/❌  | ...      | ...      |

**Interpretation:**
- `MAE@MVTS >> MAE@best` → MVTS picked a significantly worse k; likely the sparse-data bug
- MVTS k ≫ best k → MVTS over-estimates transit times → too conservative → low assignment rate
- MVTS k ≪ best k → MVTS under-estimates transit times → over-assigns → high TPQ
- `n (samples)` very low (< 5 in a 15-min window) → sparse data bug is active (event_name filter)

**Root cause confirmation**: If MVTS's own logs show it queried with `event_name='goto_completed'`
as a field filter and got far fewer rows than the full table contains, the bug is confirmed.

Grep MVTS logs:
```bash
ssh JumpServer "kubectl exec mvts-0 -n stpbulk-cluster-<env>-greymatter -- bash -c \
  'grep -i \"dynamic multiplier\|IDC multiplier\|calculating.*k\|MSU_TO_PPS\" /app/data/logs/scheduler.log | tail -20'"
```

---

## Config Reference

| Config | Default | Description |
|--------|---------|-------------|
| `ENABLE_PERIODIC_IDC_MULTIPLIER_CALCULATION` | `false` | Enable dynamic k recalculation |
| `IDC_MULTIPLIER_START` | `1.0` | Grid search lower bound |
| `IDC_MULTIPLIER_END` | `5.0` | Grid search upper bound |
| `IDC_MULTIPLIER_STEP` | `0.25` | Grid search step size |
| `IDC_MULTIPLIER_DEFAULT` | `1.0` | Static fallback when dynamic calc is off |

## MSU_TO_PPS Leg Path Types

All `original_path_type` values that count as the MSU-to-PPS leg:
- `storable_to_pps_queue`
- `relay_io_point_to_pps_entry`
- `relay_io_point_to_pps`
- `non_storable_to_relay_io_point`
- `non_storable_to_pps`
- `relay_io_point_to_conveyor_entry`
- `relay_storable_to_conveyor_entry`
- `storable_to_relay_io_point`

## InfluxDB Tag vs Field gotcha

In `task_cycle_times`:
- **Tags** (filterable, indexable): `installation_id`, `original_path_type`, `pps_id`, `path_type`, `fulfilment_area`, `host`, `butler_id`
- **Fields** (not tags): `original_idc_time`, `total_path_travel_time`, `event_name`, `idc_time_ms`, `path_travel_time_ms`

`event_name` is a **field**, not a tag. Using `WHERE event_name='goto_completed'` as a tag filter
silently returns sparse results (or 0 rows) — this is the MVTS bug causing wrong k values.
