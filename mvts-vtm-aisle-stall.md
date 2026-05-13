---
description: >
  MVTS VTM Aisle Assignment Stall & PPS Starvation — diagnoses why VTM bots show "No eligible aisle found"
  even with tasks in the PS, and traces which PPS are starved as a result.
  TRIGGER when: all VTM bots log "No eligible aisle found" every cycle, aislesWithTasks is empty or very narrow,
  PPS throughput has dropped, relay points in some aisles are all blocked, or a single bot is stuck at max
  task capacity for >30 min.
  SKIP: bots have assigned_aisle pointing to other bots' aisles (cyclic) — use mvts-vtm-deadlock instead.
---

# MVTS VTM Aisle Assignment Stall & PPS Starvation

**Symptom**: All VTM bots log "No eligible aisle found" every cycle. `Total VTM Bots assigned: 0`.
Unlike a deadlock, bots have `assigned_aisle: null` and are `ready` — no cycle, just nothing to assign.

---

## Prerequisites

Resolve namespace, confirm pod is `Running`, and save the PS. See `mvts-debug` for access patterns.

```bash
# Save latest PS
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'grep \" Message:\" /app/data/logs/scheduler.log | tail -1 | sed \"s/.*Message: //\" > /tmp/ps.json && echo done'"
```

---

## Step 1 — Read the TAM strategy init log

This single log line tells you everything about what MVTS sees for VTM assignment:

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'grep \"TAM strategy init\" /app/data/logs/scheduler.log | tail -3'"
```

**What to look for:**

```
TAM strategy init: 31 bots, 3 old assignments,
  aislesWithTasks=[30],        ← only aisles MVTS will assign bots to
  aislesOccupied=[30],         ← aisles already held by a bot
  aislesAvailable=[1,2,5,...]  ← aisles a bot could move to
```

Key signals:
| Signal | Meaning |
|--------|---------|
| `aislesWithTasks=[]` | No VTM store/relay tasks at all — upstream GMC issue |
| `aislesWithTasks=[X]` narrow | Only one or two aisles have work; others are blocked or empty |
| `aislesOccupied` ⊇ `aislesWithTasks` | Every aisle with tasks is already held by a bot at max capacity |

Also check the greedy assignment line immediately after:
```
Greedy VTM assignment start: N eligible bots, M store-to-relay tasks, K relay-to-store totes, P pinned, Q deassigned
```
If `M + K = 0` → no VTM work exists. If `M + K > 0` but assignments = 0 → bot at threshold is blocking.

---

## Step 2 — Find stuck bots (at threshold, not progressing)

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'grep \"reached its assignment threshold\" /app/data/logs/scheduler.log | tail -5'"
```

A bot hitting threshold every cycle is stuck — it has tasks from a previous cycle that GMC hasn't completed.

Identify the stuck bot and its tasks:
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \".ranger_list[] | select(.version == \\\"VTM_QT_C56_S_2D\\\" and .status == \\\"processing\\\") | {id, current_aisle: .current_aisle_info.aisle_id, available_at_time, tasks: [.ranger_schedule[] | {task_subtype, task_status, tote: .transport_entity_id}]}\" /tmp/ps.json'"
```

Check when the bot last made a NEW assignment (not the same repeated task):
```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'grep \"Total VTM Bots assigned\" /app/data/logs/scheduler.log | grep -v \": 0\" | tail -5'"
```

If the last non-zero assignment was >30 min ago, the bot is stalled.

---

## Step 3 — Confirm stall duration via InfluxDB

```influxql
-- When did the stuck bot last enter its aisle?
SELECT * FROM mvts_relay_vtm_bot_aisle_switches
WHERE time > now() - 12h
AND "InstallationId" = '<installation_id>'
AND "bot_id_tag" = '<bot_id>'
ORDER BY time ASC
```

If the only switch was hours ago with `from_aisle < 0` (dummy/transit) → bot has been stuck in that aisle since.

```influxql
-- Last actual assignments for the stuck bot
SELECT * FROM mvts_relay_vtm_bot_assignment_details
WHERE time > now() - 12h
AND "InstallationId" = '<installation_id>'
AND bot_id = <bot_id>
ORDER BY time DESC
LIMIT 10
```

If the same `task_chain` totes appear repeatedly → tasks are looping, not completing.
If there are no rows for >1h → GMC stopped acknowledging completion entirely.

---

## Step 4 — Relay point status per aisle

Count available / occupied / blocked / reserved relay points across all aisles:

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \"[.relay_point_list[] | {aisle: .aisle_info.aisle_id, status, reserved: (.reserving_tote_id != null)}] | group_by(.aisle) | map({aisle: .[0].aisle, total: length, available: (map(select(.status == \\\"available\\\")) | length), occupied: (map(select(.status == \\\"occupied\\\")) | length), blocked: (map(select(.status == \\\"blocked\\\")) | length), reserved: (map(select(.reserved)) | length)}) | sort_by(.aisle)\" /tmp/ps.json'"
```

**Interpretation:**
| Pattern | Meaning |
|---------|---------|
| All relay points `blocked`, no tote IDs | Aisle marked blocked at GMC level — relay IO points physically inaccessible or GMC bug |
| `occupied` relay points with same totes for hours | VTM task stuck executing; GMC not completing the store/pickup |
| `reserved` but 0 `occupied` | Tote reserved at relay point, waiting for HTM pickup — healthy |

Aisles where all relay points are `blocked` with `tote: null` = **fully dark aisles** — VTM cannot operate there at all regardless of task count.

---

## Step 5 — Tote inventory per aisle

For each aisle of interest, find all totes and whether they're at a relay point (ready for HTM) or stored in the rack (needs VTM first):

```bash
# Totes at relay IO points in an aisle (relay-side)
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \"[.relay_point_list[] | select(.aisle_info.aisle_id == <AISLE_ID> and .reserving_tote_id != null) | {relay_point: .id, tote: .reserving_tote_id, status}]\" /tmp/ps.json'"

# Totes in HTM task list for an aisle (stored in rack, needing VTM to relay)
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \"[.task_list[] | select(.aisle_info.aisle_id == <AISLE_ID>) | {tote: .transport_entity_id, pps: .destination_id, task_subtype, status, assigned_ranger: .assigned_ranger_id}]\" /tmp/ps.json'"
```

**Tote flow in relay-bulk:**
```
[Stored in rack] → VTM (relay_to_storable or store_to_relay) → [At relay IO point] → HTM (storable_to_pps) → PPS
```

A tote with a `storable_to_pps` task but sitting in the rack = HTM cannot execute it until VTM moves it to a relay IO point.

---

## Step 6 — PPS starvation analysis

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \"
  {
    all_pps: ([.pps_list[].id] | sort),
    active_pps: ([.task_list[].destination_id] | unique | sort),
    per_pps: ([.task_list[] | {pps: .destination_id, aisle: .aisle_info.aisle_id}]
      | group_by(.pps)
      | map({
          pps: .[0].pps,
          total: length,
          from_blocked: (map(select(.aisle == <BLOCKED_AISLE_1> or .aisle == <BLOCKED_AISLE_2>)) | length),
          from_other: (map(select(.aisle != <BLOCKED_AISLE_1> and .aisle != <BLOCKED_AISLE_2>)) | length)
        })
      | sort_by(.pps))
  }\" /tmp/ps.json'"
```

Replace `<BLOCKED_AISLE_N>` with the aisles identified in Step 4.

**What to report:**
- PPS with 0 tasks → completely starved, no work in any aisle for them
- PPS where `from_blocked / total > 0.5` → >50% of their work is stuck
- PPS with `from_other > 0` but still `to_be_assigned` → secondary bottleneck (e.g. aisle 30 VTM stall)

---

## Step 7 — Identify which tasks can actually execute now

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'jq \"[.task_list[] | select(.aisle_info.aisle_id != <BLOCKED_AISLE_1> and .aisle_info.aisle_id != <BLOCKED_AISLE_2>)] | group_by(.aisle_info.aisle_id) | map({aisle: .[0].aisle_info.aisle_id, count: length, tasks: [.[] | {tote: .transport_entity_id, pps: .destination_id, status}]})\" /tmp/ps.json'"
```

Cross-check: for each non-blocked aisle task, confirm the tote has an occupied/reserved relay point.
If yes → HTM can pick it now. If no → needs VTM first.

---

## Root cause patterns

| Pattern | Diagnosis | Owner |
|---------|-----------|-------|
| All relay points `blocked` (no tote), `aislesWithTasks` excludes that aisle | Relay IO points physically or logically blocked at GMC level | GMC team |
| Single bot stuck `processing` same totes for >30 min | GMC not completing the VTM store/pick task — tote may be physically stuck or task state corrupted | GMC team |
| `aislesWithTasks=[]`, 0 store-to-relay tasks in PS | GMC not generating VTM tasks at all — no items at relay IO points or VTM task generation paused | GMC team |
| `VTM_TASK_ASSIGNMENT_THRESHOLD` too high | Aisles with fewer tasks than threshold are skipped even when work exists | Config change |

---

## Resolution

**MVTS cannot self-resolve blocked relay points or stuck GMC tasks.**

Escalate to GMC team with:
1. The blocked aisles and relay point IDs (`TLOC_*`)
2. The stuck bot ID, aisle, and tote IDs from its `ranger_schedule`
3. The timestamp when the stall began (from InfluxDB aisle switch)
4. PPS starvation summary (which PPS, what % blocked)

---

## Key config flags

| Flag | Effect on this failure mode |
|------|-----------------------------|
| `ENABLE_VTM_TASK_REASSIGNMENT` | `false` = no pins, every cycle reassigns from scratch; stuck bot at threshold blocks the aisle every cycle |
| `VTM_TASK_ASSIGNMENT_THRESHOLD` | Min tasks in aisle before MVTS sends a bot; raising this makes starvation worse |
| `STATIC_BOT_AISLE_MAPPING_OVERRIDE` | Pins specific bots to specific aisles; if mapped aisle is blocked, that bot is permanently idle |

```bash
ssh JumpServer "kubectl exec mvts-0 -n <namespace> -- bash -c \
  'curl -s localhost:8080/mvts/config/all | python3 -m json.tool | grep -iE \"vtm|threshold|static|aisle\"'"
```

---

## Notes

- `aislesWithTasks` in the TAM log reflects VTM store/relay task counts, NOT the HTM `task_list` size. An aisle can have 22 HTM tasks but still show 0 in `aislesWithTasks` if no VTM relay-IO work is pending.
- Blocked relay points with `tote: null` are distinct from occupied relay points. Blocked = GMC-side lock. Occupied = tote physically present.
- A bot stuck at threshold is NOT a deadlock — it has real tasks, they're just not completing at GMC.
- PPS 132–135 tend to be most sensitive because they draw heavily from a small number of aisles.
